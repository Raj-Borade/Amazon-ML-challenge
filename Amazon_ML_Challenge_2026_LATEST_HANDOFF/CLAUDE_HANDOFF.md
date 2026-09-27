# CLAUDE_HANDOFF.md — Read this first

This is a handoff from Claude to Codex for the Amazon ML Challenge 2026
Business Entity Resolution project. Stage 1 (data recon) and Stage 2
(normalization + blocking) are complete and measured. Stage 3 (feature
engineering + matching model) has not been started.

**Read order for Codex**: this file → `CODEX_STATE.md` (exact resume
spec, commands, next task) → `TASK_BOARD.md` (status checklist) →
`PROJECT_STATE.md` (full history/decisions) → `ARCHITECTURE.md` (design
rationale + full measured cap-selection history) → `DATA_PROFILE.md`
(Stage 1 raw statistics, if needed for context).

## What was actually done
1. **Stage 1**: inspected all 7 supplied TSVs (2.2M–5.3M rows each),
   measured schema, missingness, country distribution (train: US/India
   only; test: adds France, ~15% of test S1 — open-set requirement is
   real, confirmed empirically not just per the spec), duplicate/name-
   ambiguity rates, match-cardinality distribution, S2/S3 evidence
   composition. Full numbers in `DATA_PROFILE.md`.
2. **Stage 2**: built a normalization module, a reproducible 30k-entity
   stratified dev sample, a full-corpus (10.32M record) disk-backed
   SQLite blocking index, 6 blocking channels, and a full measurement +
   miss-analysis + iteration cycle. Final measured recall: **93.81%
   micro / 93.79% macro** on the dev sample, up from an initial 86.8% —
   the improvement came from a specific, measured finding (see below),
   not guesswork.

## The two things most worth knowing before you touch the code
1. **A real normalization bug was found and fixed mid-stage**: an early
   version of the diacritic-stripper and tokenizer mistreated Devanagari/
   Bengali combining marks (matras, virama) as either decorative Latin
   accents or punctuation, silently shredding Indic-script business names
   into single-character fragments (e.g. "प्राइवेट" → `['प','र','इव','ट']`).
   This was caught empirically (the very first token-frequency table had
   single Devanagari characters as the most "frequent" tokens in the
   whole 10M-record corpus — that's what gave it away) and fixed. Two
   permanent regression tests now guard against it recurring
   (`tests/test_normalize.py`). If you're touching normalization code at
   all, run these tests before and after.
2. **The recall improvement from 86.8% → 93.81% came from raising a
   token document-frequency cap (500 → 1500)**, after miss analysis
   showed the two largest fixable miss buckets (41% of sampled misses
   combined) were caused by genuinely specific, matching words
   ("coastal", "litchfield", "physical", "therapy") being just common
   enough to fall outside the original cap and get excluded from the
   index entirely. This is documented with the full before/after numbers
   in `ARCHITECTURE.md` §6 — it was a measured, justified change, not an
   arbitrary parameter tweak.

## What NOT to do
- Don't re-run Stage 1 or Stage 2 experiments. The numbers in
  `PROJECT_STATE.md`/`ARCHITECTURE.md` are real and reproducible (exact
  commands are in `CODEX_STATE.md` §6) — re-running them should get you
  the same numbers, not new ones.
- Don't change `src/preprocessing/normalize.py` or
  `src/blocking/strategies.py` logic without running the existing test
  suites first and after (26 tests total, `tests/test_normalize.py` +
  `tests/test_blocking.py`).
- Don't assume you have more disk than 3.4GB free just because the code
  doesn't crash locally — the original build environment was that tight,
  and the biggest artifact (the 3.7GB train blocking index) was
  deliberately left OUT of this handoff package for exactly that reason.
  See `CODEX_STATE.md` §2/§6 for what you need to regenerate it.
- Don't start Stage 3 (feature engineering / matching model) without
  first building the test-corpus blocking index (Stage 2's index only
  covers train_source2+3) — that's the literal next task, spelled out in
  `CODEX_STATE.md` §4.

## Package contents note
This ZIP does NOT contain the raw challenge datasets or the large
generated artifacts (the 3.7GB SQLite index, the 558MB normalized text
cache). It contains everything needed to REGENERATE them
deterministically, plus every small measured result (JSON reports, the
dev sample, miss-analysis outputs) so none of Stage 2's actual findings
are lost — only the large, mechanically-regenerable binary artifacts are
excluded. See `CODEX_STATE.md` §2 for exact dataset paths/filenames
needed and §6 for the regeneration commands.
