---
phase: 01-data-ingestion-cleaning
verified: 2026-05-22T00:00:00Z
status: passed
score: 6/6 must-haves verified
overrides_applied: 0
---

# Phase 1: Data Ingestion & Cleaning Verification Report

**Phase Goal:** A clean, temporally-ordered `matches` DataFrame exists with canonical team names and a fixed train/test split boundary
**Verified:** 2026-05-22
**Status:** passed
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths (from PLAN frontmatter must_haves + ROADMAP success criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Running `uv sync` installs pyarrow, seaborn, pytest, nbmake into .venv | VERIFIED | `python -c "import pyarrow, seaborn, nbmake, pytest"` exits 0; pyarrow 24.0.0, seaborn 0.13.2, pytest 9.0.3 |
| 2 | Executing `notebook_data.ipynb` end-to-end finishes with no cell errors | VERIFIED | `pytest --nbmake notebook_data.ipynb -x` exits 0 in 2.26s; all 9 code cells have output |
| 3 | After execution, `dados/matches_train.parquet` and `dados/matches_test.parquet` exist on disk | VERIFIED | Both files exist; train=8025 rows, test=1140 rows; schema matches 11-column contract; temporal separation confirmed (train max 2022-11-13, test min 2023-04-15) |
| 4 | The notebook prints a class distribution where HomeWin is reported as 49.6% | VERIFIED | Computed from parquet: `(result=='HomeWin').mean() = 0.4965`; EDA cell asserts `abs(rate - 0.496) < 0.005` and passes; literal `0.496` token in cell 9 source |
| 5 | Top 10 Brazilian clubs each have >= 380 combined home+away match rows after normalization | VERIFIED | All 10 clubs verified: Flamengo 894, Corinthians 856, Sao Paulo 894, Fluminense 894, Santos 856, Internacional 856, Atletico-MG 855, Athletico-PR 818, Gremio 814, Palmeiras 810 |
| 6 | `pytest --nbmake notebook_data.ipynb -x` exits with code 0 | VERIFIED | `1 passed in 2.26s` — directly observed |

**ROADMAP Success Criteria cross-reference:**

| # | ROADMAP Success Criterion | Status | Evidence |
|---|--------------------------|--------|----------|
| SC-1 | `campeonato-brasileiro-full.csv` loads without error and all 9,165 rows are accounted for | VERIFIED | Cell 2 asserts `len(df) == 9165`; nbmake passes |
| SC-2 | Team name normalization; top clubs 380+ rows after normalization | VERIFIED | Cell 6 applies `canonical_names.replace()` before any groupby; loop asserts 380+ for all 10 clubs |
| SC-3 | Data sorted by date; fixed year cutoff; non-overlapping train/test | VERIFIED | Cell 3 asserts `is_monotonic_increasing`; Cell 7 asserts 8025/1140/no-overlap/temporal-separation |
| SC-4 | EDA cell confirms observed home win rate and class distribution | VERIFIED | Cell 9 has embedded `image/png` output; asserts 49.6% home win rate; prints value_counts |

**Score:** 6/6 truths verified

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `pytest.ini` | pytest config enabling nbmake plugin | VERIFIED | Contains `[pytest]` header, `addopts = --nbmake -x`, `testpaths = .`, `norecursedirs = .venv .git __pycache__ .planning` |
| `pyproject.toml` | Project dependencies including pyarrow, seaborn, pytest, nbmake | VERIFIED | All four packages listed: `pyarrow>=24.0.0`, `seaborn>=0.13.2`, `pytest>=9.0.3`, `nbmake>=1.5.5` |
| `notebook_data.ipynb` | Phase 1 data pipeline notebook (9 cells) | VERIFIED | 9 code cells + 10 markdown cells; all cells executed with outputs; EDA cell has embedded `image/png` |
| `dados/matches_train.parquet` | 8025 rows, 2003-2022, 11-column schema | VERIFIED | 8025 rows; columns `['ID', 'date', 'season', 'round', 'home_team', 'away_team', 'home_score', 'away_score', 'home_state', 'away_state', 'result']`; 0 nulls; `date` dtype `datetime64[ns]` |
| `dados/matches_test.parquet` | 1140 rows, 2023-2025, same schema | VERIFIED | 1140 rows; same 11-column schema; 0 nulls; `date` dtype `datetime64[ns]` |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `notebook_data.ipynb` | `dados/archive/campeonato-brasileiro-full.csv` | `pd.read_csv` | WIRED | Literal `pd.read_csv('dados/archive/campeonato-brasileiro-full.csv')` in cell 2 source |
| `notebook_data.ipynb` | `dados/matches_train.parquet` | `DataFrame.to_parquet(engine='pyarrow')` | WIRED | `to_parquet('dados/matches_train.parquet'` with `engine='pyarrow'` in cell 8 |
| `notebook_data.ipynb` | `dados/matches_test.parquet` | `DataFrame.to_parquet(engine='pyarrow')` | WIRED | `to_parquet('dados/matches_test.parquet'` with `engine='pyarrow'` in cell 8 |
| `pytest.ini` | `notebook_data.ipynb` | nbmake plugin discovery | WIRED | `addopts = --nbmake -x` in `pytest.ini`; `pytest --nbmake notebook_data.ipynb -x` exits 0 |

---

## Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|--------------------|--------|
| `matches_train.parquet` | All 11 columns | `campeonato-brasileiro-full.csv` via `pd.read_csv` | Yes — real DB query equivalent: CSV with 9,165 rows, row counts asserted | FLOWING |
| `matches_test.parquet` | All 11 columns | Same CSV, year >= 2023 slice | Yes — 1,140 rows produced from real data | FLOWING |
| EDA cell (cell 9) | `result` column distribution | `train_df` / `test_df` in-memory from notebook execution | Yes — computes `value_counts()` from live DataFrame; asserts 49.6% | FLOWING |

---

## Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| `pytest --nbmake notebook_data.ipynb -x` exits 0 | `pytest --nbmake notebook_data.ipynb -x` | `1 passed in 2.26s` | PASS |
| Parquet files have correct row counts and schema | `python -c "pd.read_parquet(...); assert len(t)==8025..."` | train=8025, test=1140, schema OK, no nulls, no overlap | PASS |
| Top 10 clubs have 380+ rows | Python check on parquet concat | All 10 clubs pass (min: Palmeiras 810) | PASS |
| HomeWin rate is 49.6% (+/- 0.5pp) | Computed from parquet | 49.65% — within tolerance | PASS |
| No forbidden patterns in notebook | `grep KFold/StratifiedKFold/random_state/shuffle=True` | 0 occurrences each | PASS |
| `df.dropna(` not used as bare call | context search | 1 occurrence in a comment explaining why NOT to use it — not executable code | PASS |

---

## Probe Execution

No probe scripts declared or applicable (non-migration phase; nbmake serves as the probe).

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| DAT-01 | 01-01-PLAN.md | Load and parse `campeonato-brasileiro-full.csv` (9,165 matches, 2003-2025) | SATISFIED | Cell 2 asserts `len(df) == 9165` and verifies 17-column list; nbmake passes |
| DAT-02 | 01-01-PLAN.md | Normalize team names to canonical mapping; top clubs 380+ rows | SATISFIED | Cell 6 applies `canonical_names.replace()` before any groupby; 10-club 380+ assertion passes |
| DAT-03 | 01-01-PLAN.md | Temporal train/test split — sorted by date, fixed cutoff, no random shuffle | SATISFIED | Cell 7 asserts 8025/1140 rows, no index overlap, chronological separation; no `KFold`/`shuffle` in notebook |

**Orphaned requirements check:** No requirements mapped to Phase 1 in REQUIREMENTS.md that are absent from the plan. DAT-01, DAT-02, DAT-03 are the only Phase 1 requirements. All covered.

---

## CONTEXT.md Decision Compliance (D-01 through D-05)

| Decision | Requirement | Status | Evidence |
|----------|-------------|--------|----------|
| D-01: Deliverable is `notebook_data.ipynb` | Dedicated data notebook | HONORED | File exists, 9 code cells, executed |
| D-02: Two parquet files at `dados/matches_train.parquet` and `dados/matches_test.parquet` | Shared contract for downstream phases | HONORED | Both files exist with correct schema |
| D-03: Cutoff year 2023; train <= 2022 (8025), test >= 2023 (1140) | No shuffle/random split | HONORED | Cell 7 implements exact year cutoff; assertions pass |
| D-04: `result` column with string labels `HomeWin`/`Draw`/`AwayWin` | Consistent encoding across all model notebooks | HONORED | `set(result.unique()) == {'HomeWin', 'Draw', 'AwayWin'}` in both parquet files |
| D-05: Manual canonical-names dict before any groupby | Normalization guard | HONORED | Cell 6 applies `canonical_names.replace()` (currently empty dict) before 380+ assertion |

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `notebook_data.ipynb` | Cell 10 (code cell 5), line 14 | `df.dropna(` token | INFO | Comment-only: "NÃO por df.dropna()" — explains why it is forbidden. Not executable. No functional impact. |

No debt markers (TBD, FIXME, XXX) found in modified files. No stub patterns found. No empty implementations.

---

## Human Verification Required

None. All phase behaviors are covered by automated checks (inline notebook assertions + nbmake execution). No visual appearance or real-time behavior items.

---

## Gaps Summary

No gaps. All 6 must-haves verified. All ROADMAP success criteria satisfied. All 3 requirement IDs covered. All 5 context decisions honored. `pytest --nbmake notebook_data.ipynb -x` is green.

---

_Verified: 2026-05-22T00:00:00Z_
_Verifier: Claude (gsd-verifier)_
