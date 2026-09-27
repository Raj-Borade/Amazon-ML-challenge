# CODEX_STATE.md — Resume Spec for Codex

Purpose: everything a coding agent needs to resume this project WITHOUT
re-deriving anything already measured. If a number appears here, it was
actually measured in this project (see `PROJECT_STATE.md`/`ARCHITECTURE.md`
for the full derivation) — do not re-measure it as a first step; trust it
and build on it. Re-run only what §4 (Immediate Next Task) asks for.

## 1. Environment assumptions (measured on the machine that produced this
handoff — verify on yours, don't assume identical)
- Python 3.12
- Packages already used and required: `pandas`, `scikit-learn`, `flask`,
  `rapidfuzz`, `pytest` (all pip-installable; `rapidfuzz`/`pytest` needed
  `pip install --break-system-packages <pkg>` in the original sandbox —
  adjust for your environment).
- Original resource envelope: ~3.9GB RAM, 1 CPU core, ~7.7GB disk total.
  All code in `src/` was deliberately written to be memory-safe under
  this envelope (streaming/chunked reads, disk-backed SQLite index,
  never loading the 5M+/row source files fully into pandas at once). If
  your environment has more resources, the code still works — it just
  doesn't need to be as careful; do not assume the reverse (do not run
  the existing scripts on a MORE constrained environment without re-
  checking §5 numbers below).

## 2. Dataset status — READ THIS BEFORE RUNNING ANYTHING
**The raw challenge datasets are NOT included in this handoff package**
(see `CLAUDE_HANDOFF.md` for why — disk size). You must obtain them
separately and place them at these EXACT relative paths for every
existing script to work unmodified:

```
dataset/train/train_source1.tsv        (2,206,821 rows + header)
dataset/train/train_source2.tsv        (5,034,616 rows + header)
dataset/train/train_source3.tsv        (5,285,603 rows + header)
dataset/train/train_ground_truth.tsv   (2,206,821 rows + header)
dataset/test/test_source1.tsv          (1,732,544 rows + header)
dataset/test/test_source2.tsv          (4,887,273 rows + header)
dataset/test/test_source3.tsv          (5,082,316 rows + header)
```
All are TSV (`sep="\t"`), UTF-8, 4 columns each (2 for ground truth):
`entity_id, business_name, business_address, country` /
`source1_entity_id, matched_entity_ids`. Row counts above are exact and
were verified in Stage 1 (`DATA_PROFILE.md` §2) — use them to confirm you
have the right files (`wc -l` should match, off by the header row).

Original upload filenames (for provenance only, not needed to run
anything): `1790398762883_train_ground_truth.tsv`,
`1790399276579_train_source1.tsv`, `1790399486470_train_source2.tsv`,
`1790399739794_train_source3.tsv`, `1790399939595_test_source1.tsv`,
`1790399999809_test_source2.tsv`, `1790400265160_test_source3.tsv`
(see `DATA_PROFILE.md` §0 for the full mapping).

## 3. What's already built and verified (do not redo)
| Artifact | Path (in package) | Status |
|---|---|---|
| Normalization module | `src/preprocessing/normalize.py` | Done, 18 tests pass |
| Stratified dev sample | `artifacts/dev_sample_ground_truth.tsv` + `dev_sample_manifest.json` | Done, seed=42, 30,000 S1 entities |
| Blocking channels | `src/blocking/strategies.py` | Done, 6 channels, 8 tests pass |
| Cache-build pipeline | `src/blocking/build_cache.py` | Done (code only — cache files themselves NOT in package, see §2/§6) |
| Index-build pipeline | `src/blocking/build_index.py`, `extend_index.py` | Done (code only — the 3.7GB `.db` itself NOT in package) |
| Evaluation harness | `src/blocking/evaluate.py` | Done |
| Miss analysis | `src/blocking/miss_analysis.py`, `miss_classify.py` | Done, results in `artifacts/miss_analysis_report.json` |
| Token document-frequency tables | `artifacts/cache/token_df_train_source2.json`, `artifacts/cache/token_df_train_source3.json` | Included (43MB total) — lets you rebuild the index without re-running `build_cache.py`'s normalization pass, PROVIDED you also regenerate `artifacts/cache/norm_*.tsv.gz` (see §6) |

## 4. Immediate next task (do this first)
**Decide disk strategy, then build the blocking index for the TEST
corpus.** This is blocked on disk space, not on any missing code:

1. Obtain the raw datasets (§2) and place them at the documented paths.
2. Free disk if needed: the train `blocking_index.db` (3.7GB) is fully
   regenerable from `artifacts/cache/*.tsv.gz` (which you'll have
   regenerated anyway once you run `build_cache.py` — see §6), so it is
   safe to delete `artifacts/blocking_index.db` if disk is tight, AFTER
   confirming you can rebuild it from the same commands in §6.
3. Run the exact same pipeline (`build_cache.py` → `build_index.py` →
   `extend_index.py`, commands in §6) against `test_source2.tsv` /
   `test_source3.tsv` instead of the train files. **Do not change any
   parameters** (same `--max-token-df 500` then extend to 1500) — the
   cap values were measured/justified against the TRAIN corpus token
   distribution; if the test corpus's distribution differs meaningfully,
   that's worth noting, but the default is to reuse the same values
   unless you have a specific measured reason not to (that would be new
   research, out of scope per the handoff instructions — flag it, don't
   silently change it).
4. Sanity-check the resulting test index the same way Stage 2 did for
   train (spot-query a few known entity_ids, confirm row counts match
   §2's table).

**Do NOT** re-run the train-side blocking evaluation, re-run miss
analysis on train, or change `src/preprocessing/normalize.py` /
`src/blocking/strategies.py` logic — those are considered DONE per the
explicit instruction not to redo Stage 2.

## 5. Measured results to build on (do not re-derive — see
`PROJECT_STATE.md` §9 and `ARCHITECTURE.md` §6 for full derivation)
- Full 10,320,219-record (train_source2+3) index, 6 blocking channels
  combined, on the 30k dev sample:
  **93.81% micro recall / 93.79% macro recall**
  avg 2,397 candidates/entity (median 2,189, p90 4,769, p95 5,636,
  p99 7,409, max 12,682), 240.2s for 30,000 queries (~125 queries/sec).
- Only 3.68% of dev-sample entities-with-matches (1,043/28,325) had ZERO
  recovered true matches.
- Token document-frequency cap: 500 initially, raised to 1500 after miss
  analysis (justified, measured trade-off in ARCHITECTURE.md §6).
- `street_number` blocking channel MUST stay capped at its own df≤500 —
  an uncapped version produced up to 189,867 candidates for common short
  numbers. This is implemented in `street_number_block()` in
  `strategies.py`; do not remove the cap.

## 6. Exact commands (run in this order, from the project root, once
datasets are in place)
```bash
# 1. normalize + cache (streaming, ~250s per source on the original
#    1-core/3.9GB environment; scale expectation to your hardware)
python3 -m src.blocking.build_cache --source test_source2
python3 -m src.blocking.build_cache --source test_source3
#   NOTE: build_cache.py currently hardcodes reading from dataset/train/
#   (see DATASET_DIR + "train" in the script). For the test corpus you
#   must either (a) pass test_source2/test_source3 paths explicitly by
#   adjusting the path construction, or (b) symlink/copy test files into
#   a parallel "train"-named path Pattern before running. This is a real,
#   small code adjustment Codex needs to make -- it was not needed in
#   Stage 2 because only the train corpus was indexed. Keep the change
#   minimal (parameterize the directory, don't rewrite the script).

# 2. build index (cap=500) then extend to cap=1500
python3 -m src.blocking.build_index --max-token-df 500
python3 -m src.blocking.extend_index   # old_cap=500, new_cap=1500 (defaults)

# 3. tests (should all still pass, unchanged)
python3 tests/test_normalize.py
python3 tests/test_blocking.py
# or: python3 -m pytest tests/ -v

# 4. (train-side only, for reference/reproduction, NOT required for the
#    next task) reproducible dev sample + evaluation + miss analysis:
python3 -m src.data.sampling --n 30000 --seed 42
python3 -m src.blocking.evaluate
python3 -m src.blocking.miss_analysis
python3 -m src.blocking.miss_classify
```

## 7. Things that will silently break if you're not careful
- `src/blocking/build_cache.py`, `build_index.py`, `extend_index.py` all
  use **relative paths** anchored on `os.path.dirname(__file__)` — run
  them as `python3 -m src.blocking.<module>` from the project root, not
  as standalone scripts from another directory.
- The SQLite index file path is hardcoded to
  `artifacts/blocking_index.db` in both `build_index.py` and
  `extend_index.py` — building a SEPARATE test index means either
  renaming/moving the train index first (to avoid `build_index.py`'s
  `os.remove(db_path)` deleting it) or passing a different `db_path`
  explicitly (the function accepts one; the CLI entrypoint doesn't
  expose it yet — small addition needed).
- `evaluate.py`'s `TOTAL_CORPUS_SIZE` constant is hardcoded to the TRAIN
  row counts (5,034,616 + 5,285,603) — if you reuse this harness against
  the test corpus, update that constant to the test row counts
  (4,887,273 + 5,082,316) or the reduction-ratio metric will be wrong
  (recall/candidate-count themselves are unaffected).
- Ground truth (`train_ground_truth.tsv`) is never touched by the
  indexing pipeline — verified in `ARCHITECTURE.md` §8. The test corpus
  has no ground truth file at all, so `evaluate.py`/`miss_analysis.py`
  cannot run against test data as-is (no true-match labels to score
  against) — this is expected, not a bug.

## 8. Codex continuation checkpoint (2026-09-26)

- The supplied `Amazon_ML_Challenge_2026_Stage2_Handoff.zip` was extracted
  intact into this project directory. Its Stage 2 source, tests, state
  documents, and small reproducibility artifacts are present; raw TSVs and
  the deliberately omitted large caches/index are not.
- The untouched baseline command `python -m pytest tests -v` could not run:
  this host has no `python`, `python3`, `py`, package manager, or configured
  bundled Python runtime. An approved attempt to obtain an isolated runtime
  was rejected by the execution sandbox before downloading anything.
- Therefore no Stage 3 source or test change has been made. Do not treat the
  Stage 2 suite as re-verified on this host until a Python runtime and the
  pinned dependencies in `requirements.txt` are available.
- Exact resume command after environment repair:
  `python -m pytest tests -v` from the project root. Only after that baseline
  passes may Stage 3 preprocessing changes begin.

## 9. Stage 3 completion checkpoint (2026-09-26)

- The test-runtime blocker in §8 is resolved for this task by using
  `C:\Users\rajbo\anaconda3\python.exe` directly (the launcher is not on
  the task shell's PATH). The untouched baseline passed 26/26 tests.
- Stage 3 extended `src/preprocessing/normalize.py` additively and created
  `tests/test_preprocessing.py`; no blocking source, raw dataset, cache,
  index, or measured Stage 2 artifact was modified.
- The complete suite now passes **39/39** via
  `C:\Users\rajbo\anaconda3\python.exe -m pytest tests -v` (1.06s).
  See `PROJECT_STATE.md` §15 and `ARCHITECTURE.md` §9 for the exact
  representations, rules, rationale, and warnings.
- Keep using the direct interpreter path until the environment adds it to
  PATH. Non-failing warnings: old optional `numexpr`/`bottleneck` versions
  and a non-writable pytest cache directory.
- Exact next task: stop here; later, decide disk strategy and build the test
  blocking index only after the seven raw TSVs are available. No model or
  inference task began in Stage 3.

## 10. Stage 4A execution blocker (2026-09-26)

- All seven raw TSVs are now mounted and verified at their documented paths,
  exact Stage 1 row counts and schemas included. No raw TSV was modified.
- `build_cache.py`, `build_index.py`, and `extend_index.py` now accept
  isolated split/cache/source/DB parameters, preserving Stage 2 defaults.
  The focused infrastructure test and complete suite pass **40/40**.
- The host has 18.40 GiB free on C:, adequate for the expected test artifacts;
  disk capacity is not the current blocker.
- The execution harness terminates the multi-million-row cache process at
  about 30 seconds and rejects `Start-Process` background execution. A zero-
  byte partial cache remains at `artifacts/test_cache/norm_test_source2.tsv.gz`;
  its direct deletion was also rejected by sandbox policy. It must be removed
  before the documented resume commands in `PROJECT_STATE.md` §16 run.
- Exact next task: run the four commands in `PROJECT_STATE.md` §16 from an
  unrestricted local terminal, perform SQLite integrity checks, then resume
  Stage 4A only. Stages 5–11 and release audit remain unauthorized until then.

## 11. Stage 4A complete (2026-09-27)

- Manual build artifacts are present and verified: test caches under
  `artifacts/test_cache/` and `artifacts/test_blocking_index.db`
  (3,910,246,400 bytes), built at cap 500 then extended to cap 1500.
- `src/blocking/validate_index.py --fast` passed against the real test DB:
  9,969,589 total rows, exact expected source split, verified access indexes,
  records primary key, France/India/US open-set values, and representative
  exact-name retrieval from both sources. `pytest tests -v` is **40/40**.
- Full DB scans are still not runnable in this harness (30-second limit);
  no false claim of a full `quick_check` is made. Unit/infrastructure checks
  cover NULL token removal, cache isolation, and open-set country handling.
- Stage 5 blocker: train normalized caches and `artifacts/blocking_index.db`
  are absent from the handoff package. Build them on an unrestricted shell;
  do not train against the test DB or test labels.

## 12. Stage 5 feature foundation (2026-09-27)

- Added `src/features/pair_features.py` and four focused tests. It extracts
  fixed numeric evidence only for supplied candidate pairs and remains
  streaming-safe. No matching decision/model has been implemented.
- Full suite is **44/44** via the direct Python interpreter. Existing warnings
  remain non-failing (`numexpr`, `bottleneck`, non-writable pytest cache).
- Blocker for training/validation: train caches and the train SQLite index are
  absent. Current C: free space is 12.20 GiB, but the execution harness cannot
  run the required multi-minute cache/index command. Build them manually with
  existing Stage 2 semantics; never replace them with test artifacts.
