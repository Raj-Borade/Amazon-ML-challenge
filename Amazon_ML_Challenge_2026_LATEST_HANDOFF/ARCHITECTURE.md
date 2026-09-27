# ARCHITECTURE.md — Business Entity Resolution (Amazon ML Challenge 2026)

Covers the Stage 2 preprocessing + blocking architecture. Matching model
(Stage 3+) is out of scope here.

## 1. Preprocessing architecture
`src/preprocessing/normalize.py` — pure functions, never mutate/discard
the raw string. `normalize_name`/`normalize_address` return a dataclass
carrying: `raw`, `normalized`, `tokens`, `sorted_key` (order-invariant),
plus field-specific extras (`domain_stem`, `script`, `digit_tokens`,
`street_number` for address). Callers keep the raw value; every derived
view is additive, per Task D.

Multiple representations preserved per record:
- raw name / raw address (never touched)
- normalized (case-folded, whitespace-collapsed, diacritic-stripped,
  junk-prefix-stripped, legal-suffix-canonicalized) name/address
- token list (script-aware tokenizer — see §5, bug history)
- order-invariant sorted-token key (survives component reordering)
- domain stem (when the name is a bare domain string)
- digit tokens / leading street number (script-invariant signal)

Legal-suffix table covers US (`Inc`/`LLC`/`Ltd`/`Corp`/...), India
(`Pvt`/`M/s`), and France (`SAS`/`SARL`/`EURL`/`SA`) — open-set: unknown
suffixes pass through as normal tokens rather than erroring.

## 2. Blocking architecture
Two-pass, disk-backed, built once over the full train_source2 +
train_source3 (10,320,219 records combined):

**Pass 1** (`src/blocking/build_cache.py`): streams each raw TSV row by
row (csv.reader, never `pd.read_csv` on the whole 5M-row file), normalizes
name+address, writes a compact gzip-TSV cache (`artifacts/cache/norm_
<source>.tsv.gz`) with only the derived fields, and accumulates token
document-frequency counters in memory (bounded — a few hundred KB per
source, not the corpus itself).

**Pass 2** (`src/blocking/build_index.py`, extended by
`extend_index.py`): reads the cache (cheap, no re-normalization) into a
SQLite index (`artifacts/blocking_index.db`) with:
- `records(entity_id PK, source, country, name_norm, name_sorted_key,
  addr_norm, street_number, domain_stem)` — one row per S2/S3 record
- `name_tok(token, entity_id)`, `addr_tok(token, entity_id)` — inverted
  postings, **capped by document frequency** (currently df ≤ 1500 — see
  §6 for why 500 was insufficient and 1500 was chosen)
- B-tree indexes on `token`, `name_sorted_key`, `street_number`,
  `domain_stem`, `country`

This is "rare-token blocking" implemented as the index's own admission
rule rather than a query-time stopword list: a token common enough to
exceed the cap is structurally incapable of narrowing a candidate set, so
it's never stored.

## 3. Candidate-set representation
A candidate set for one S1 query is `set[str]` of `S2-`/`S3-` prefixed
entity ids — the union of whichever blocking channels are enabled,
deduplicated (channels frequently overlap; dedup is a hash-set op, not a
separate step — tested in `tests/test_blocking.py`). No candidate pair is
materialized as a DataFrame row until the (future) feature-engineering
stage; blocking only produces id sets.

## 4. Blocking channels (final set, all implemented in
`src/blocking/strategies.py`)
| channel | mechanism | why kept |
|---|---|---|
| `exact_name` | `name_sorted_key` equality | cheap, catches suffix-position/case noise exactly |
| `name_rare_token` | union of postings for query's name tokens (capped) | main name-side recall driver |
| `addr_rare_token` | same, address tokens | stronger than name alone (address is less ambiguous — Stage 1 finding) |
| `street_number` (capped, own df ≤ 500) | exact leading-digit-run match | script-invariant; had to be capped separately after an uncapped version produced up to 189,867 candidates for common numbers like "1" |
| `domain_stem` | query name itself looks like a domain → match target domain stem | catches Stage-1-documented domain-style names |
| `squashed_name_vs_domain` | query's tokens concatenated (no spaces) → match target domain stem | reverse of the above; added after miss analysis found normal-S1-name-vs-domain-target was uncovered (~10% of sampled misses) |

Rejected/deferred channels (Task C, recorded per the failure-handling
rule, not silently dropped):
- **Character n-gram / trigram inverted index**: sized it before
  building. Because the effective character-trigram vocabulary for this
  corpus is small (a few tens of thousands of trigrams over ~10.3M
  records), per-trigram document frequency is extremely high on average
  — a flat df-capped trigram index would exclude almost everything,
  unlike word-level tokens where the tail is long. A working version
  needs top-K-rarest-trigrams-per-record indexing instead of a flat cap,
  which was not implemented this stage due to disk risk (see §6) and
  time budget. **Not rejected outright** — flagged as the top candidate
  for a future increment if typo-driven misses (~15.8% of the sampled
  miss set, see `artifacts/miss_analysis_report.json`) turn out to matter
  more once a real classifier is in place.
- **Token TF-IDF / full rarity-weighted retrieval** (rather than a hard
  df cutoff): the cap-extension experiment (§6) is a coarse approximation
  of this (raising the cutoff) and already bought +7 recall points. A
  true IDF-weighted top-K retrieval was not implemented; the blunt
  cap-raise was chosen because it reused 100% of existing code/index
  structure with no new tables, at acceptable disk cost.
- **Postal/PIN retrieval**: Stage 1 found no reliable, separately-parseable
  postal/PIN field in the data — postal-like digits are only present as
  incidental digit tokens already covered by `street_number`. Not
  implemented as its own channel.
- **Reverse S2/S3 → S1 retrieval, cross-source structure (Task H)**: the
  blocking direction (S1 query → S2/S3 candidates) is what the output
  format needs; a formal reverse index was not built this stage. No
  ground-truth leakage risk was introduced because no cross-source
  co-occurrence learned from ground truth is used at index-build time —
  the index is built purely from S2/S3 text, independent of any S1/GT
  data (verified: `build_cache.py`/`build_index.py` never open
  `train_ground_truth.tsv` or `train_source1.tsv`).

## 5. Bug found and fixed during this stage: Devanagari/Bengali shredding
An early version of `strip_diacritics` (blanket
`unicodedata.combining(ch)` filter) and `tokenize` (blanket `[^\w\s]`
regex) both mistreated Indic combining marks (matras, virama — Unicode
category M*) as either decorative Latin diacritics or punctuation. Real
effect on real data: `"प्राइवेट"` (Private) was being shredded into
`['प','र','इव','ट']` — single-character garbage tokens — which is why
the very first (pre-fix) token-frequency table had single Devanagari
*characters* among the most frequent "tokens" in the whole corpus.
Fixed by (a) restricting diacritic-stripping to the Latin Combining
Diacritical Marks block (U+0300–U+036F) only, and (b) replacing the regex
tokenizer with a Unicode-general-category grouper (L*/N*/M* = word chars).
Caught via direct inspection while building the index, not by luck —
recorded as a permanent regression test
(`test_devanagari_words_not_shredded_by_combining_marks`,
`test_bengali_script_not_shredded`).

## 6. Memory/disk strategy and the cap-1500 decision
Environment: 3.9GB RAM / 1 core / ~7.7GB disk (see PROJECT_STATE.md §2).
Everything in this stage streams row-by-row or queries via indexed SQLite
lookups; nothing holds the 10.3M-record corpus as Python objects at once.
Peak measured RAM during index build: ~0.4GB process (buff/cache grows
separately, not our budget).

Cap history (this is the concrete, measured version of Task F's
"if a cap is necessary, justify/measure/record" requirement):
1. Built the index at token df ≤ 500 (~17.6M total postings, DB size
   3.5GB). Combined-channel recall on the 30k dev sample: **86.8%**
   micro, avg 564 candidates/entity (see
   `artifacts/blocking_evaluation_results.json`).
2. `street_number` blocking, run uncapped, produced avg 21,910 / max
   189,867 candidates on a 1000-entity smoke test — common short numbers
   ("1", "10") are shared by thousands of unrelated addresses. Fixed by
   capping `street_number`'s own document frequency at ≤500 (same
   philosophy as token capping). Recall from that channel alone dropped
   from 67.5% to 14.1% standalone, but it stopped being pathological and
   the *combined* union barely changed (street_number mostly duplicated
   what name/addr token channels already found).
3. Miss analysis (Task B, `artifacts/miss_analysis_report.json`, 500
   sampled misses) found the two largest fixable buckets —
   `legal_suffix_variation` (22.2%) and `other` (18.8%), together 41% of
   sampled misses — were actually the SAME root cause: real, specific
   matching words (`coastal`, `litchfield`, `physical`, `therapy`, ...)
   sit just above df=500 and were silently excluded from the index, even
   though they're far from generic stopwords.
4. Sized a cap raise before touching the index: cap 500→1500 costs an
   estimated +7.1M postings (+40%), measured against combined df-value
   tables from the token_df caches (see `extend_index.py` docstring for
   the exact numbers). At 3.4GB free disk after the extension actually
   ran (measured, not estimated), this was judged safe with margin.
5. **Measured result on the full 30k dev sample at cap=1500** (same 5
   channels as before): **93.81% micro recall / 93.79% macro recall**
   (+7.0 points over cap=500), at **avg 2,397 candidates/entity**
   (median 2,189, p90 4,769, p95 5,636, p99 7,409, max 12,682) — up from
   avg 564 at cap=500. Runtime: 240.2s for 30,000 queries against the
   full 10.32M-record index (125 queries/sec).
6. Added `squashed_name_vs_domain` (reverse domain-name channel, §4).
   Measured incremental effect on a 5,000-entity subsample: +0.1 recall
   point (18 additional pairs recovered), essentially free in candidate
   volume (2,379.5 → 2,380.2 avg). Kept because it's justified by a real
   miss category and costs nothing measurable.

**Selected architecture for Stage 3 hand-off**: cap=1500 index + all 6
channels combined. 93.81%/93.79% recall is the current measured ceiling
on final macro F_0.5 — no downstream classifier can recover a true match
outside this candidate union.

**Explicitly not yet done** (would need another dedicated pass, deferred
per the checkpoint rule rather than rushed): a true IDF/IDF-top-K
retrieval channel, a working character-n-gram channel via top-K-rarest-
trigram indexing, and full-dataset (2.2M S1 × full corpus) feasibility
timing — see PROJECT_STATE.md §10 for the exact next actions.

## 7. Determinism
- Dev sample: fixed `random_state=42` stratified sample, reproducible via
  `python3 -m src.data.sampling --n 30000 --seed 42`.
- Index build: deterministic given the same input TSVs and cap values
  (no randomness in normalization or SQL insertion order affects the
  resulting *set* of postings, though insertion order itself is file
  order and not re-randomized).
- Miss-pair sampling: `random.seed(42)` in `miss_analysis.py`.
- All of the above verified by `tests/test_blocking.py`'s determinism
  tests (same seed → identical sample; different seed → different
  sample).

## 8. No ground-truth leakage (Task I)
`train_ground_truth.tsv` is read ONLY by `src/data/sampling.py` (to build
the dev sample) and `src/blocking/evaluate.py` /
`src/blocking/miss_analysis.py` (to score candidate recall). Neither
`build_cache.py` nor `build_index.py` nor any blocking strategy in
`strategies.py` opens the ground-truth file or the S1 file — the index is
built purely from S2/S3 text and is identical whether or not ground truth
exists. This also means the same code path is valid for TEST-set blocking
once that index is built (a separate, not-yet-built `test_source2`/
`test_source3` index — see PROJECT_STATE.md next actions).

## 9. Stage 3 robust preprocessing extension

Stage 3 deliberately extends the tested Stage 2 module rather than replacing
it. The implementation remains pure, deterministic, per-record, and usable
in a streaming loop; it neither reads datasets nor materializes pandas frames.

### Additive representations and raw preservation

`NormalizedName` and `NormalizedAddress` retain the exact raw input object
(including `None` or whitespace-only input) plus independent derived views:

- `unicode_normalized`: NFKC plus collapsed whitespace, retaining punctuation
  and script content for a minimally transformed display/comparison view.
- `comparison_text`: leading observed name junk removed, Latin diacritics
  stripped, case-folded, and whitespace-normalized. This preserves the Stage
  2 treatment of synthetic accented English noise while keeping Indic marks
  safe (the existing Devanagari/Bengali regression tests remain in force).
- `punctuation_normalized`: punctuation and symbols become token separators;
  Unicode letters/numbers/marks remain untouched.
- Existing `normalized`, `tokens`, and `sorted_key`, plus new `squashed_key`
  (character-style matching), `latin_tokens` (safe cross-script anchors), and
  `content_tokens`/`legal_suffix_tokens` on names. No representation overwrites
  raw text or forces downstream callers to discard another useful view.

`NormalizedRecord`/`normalize_record` preserve all four raw source fields and
offer explicit per-field missingness plus `missing_fields`. `iter_normalized_records`
is a generator so future multi-million-row work stays bounded to one record.

### Evidence-based rules and deliberate limits

- The Stage 2 legal map is retained. Only documented punctuation spellings
  that Stage 2 tokenization previously split (`L.L.C.`, `L.L.P.`, `P.L.L.C.`,
  `M/S`) are additionally collapsed. Marker and content token views are both
  retained because legal status may be a useful feature rather than disposable
  noise.
- The existing observed address abbreviations (Avenue/Ave, Road/Rd, units,
  apartments, and related Stage 1 forms) are retained unchanged. No new
  country-specific rule was invented.
- `address_components` recognizes comma boundaries only and intentionally
  labels no city, region, postal code, or street. `house_number` requires a
  numeric start to the first component. The less-strict existing
  `street_number` remains a blocking signal, not an asserted address fact.
- `NormalizedCountry` uses NFKC, whitespace collapse, and case-folding only.
  It has no country list, ISO mapping, or rejection path, so France and unseen
  future labels pass through as open-set strings.
- No transliteration is generated. The data profile confirms cross-script
  India records but supplies no permitted, reliable mapping; fabricated output
  would create false equivalences. Script tags, intact Unicode tokens,
  `latin_tokens`, and digit tokens are the conservative alternative.

### Compatibility and validation

Existing Stage 2 fields and behavior remain available. The only intentional
normalization refinement is canonicalizing the documented punctuation variants
above; `tests/test_normalize.py` retains all Stage 2 regression assertions.
`tests/test_preprocessing.py` adds 13 focused Stage 3 tests. Full suite result:
**39 passed, 0 failed** (Python 3.12.4 / pytest 9.1.1, 2026-09-26).

## 10. Stage 4A reusable test-index plumbing

The existing Stage 2 index format and six-channel strategy are unchanged.
To build test artifacts without overwriting train artifacts, cache build now
accepts an explicit dataset split and cache directory; index build/extension
accept explicit source prefixes, cache directory, and DB path. Defaults remain
the original train configuration. The intended test layout is
`artifacts/test_cache/` plus `artifacts/test_blocking_index.db`, with the same
df=500 build then df=1500 extension and the same SQLite schema.

`tests/test_blocking_infrastructure.py` validates this isolation with a tiny
synthetic cache/index and confirms records remain queryable across both source
prefixes, entity IDs are unique, standalone NULL address components do not
enter normalized address text, and France/India/Futureland pass through the
stored open-set country field unchanged. The complete suite result is **40
passed, 0 failed**. The full real test build is pending solely on the current
execution-harness duration restriction, not a change in blocking policy.

## 11. Stage 4A completed test-corpus index

The manual build completed the isolated test-corpus artifacts without changing
the Stage 2 retrieval policy: `artifacts/test_cache/` stores normalized
`test_source2`/`test_source3` caches and per-source token document frequencies;
`artifacts/test_blocking_index.db` stores the existing `records`, `name_tok`,
and `addr_tok` schema plus indexes for exact sorted-name, rare name/address
tokens, capped street number, domain stem, and country. The token admission
policy is exactly cap 500 followed by the measured cap-1500 extension.

The 3,910,246,400-byte DB holds 9,969,589 records (4,887,273 S2 and 5,082,316
S3). `validate_index --fast` verifies the source counts, access indexes,
primary-key enforcement, normal retrieval from both sources, and open-set
country storage. Full 3.9GB scans are intentionally not claimed: this tool
harness terminates them before completion. This does not alter the measured
Stage 2 recall; no test labels enter index build or validation.

## 12. Stage 5 pair-feature foundation

`src/features/pair_features.py` is deliberately downstream of blocking: it
accepts normalized left/right fields and candidate-channel provenance, then
returns a fixed numeric mapping. It contains no retrieval, label access,
training, threshold, or match-decision behavior. The feature groups retain
multiple complementary signals rather than collapsing text to one score:
name exact/order/character/token/content/suffix/domain; address
exact/order/character/token/components/digits/numbers; open-set country
agreement; and explicit missingness. Cross-script pairs rely on the Stage 3
safe representations rather than invented transliteration. The source is
per-pair and iterator-friendly, so later sampled candidate processing can be
bounded in memory. Four new unit tests raise the suite to **44 passed**.
