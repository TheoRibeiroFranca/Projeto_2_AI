---
phase: 01-data-ingestion-cleaning
plan: 01
subsystem: data-pipeline
tags: [data-ingestion, pandas, parquet, temporal-split, notebook, nbmake]
dependency_graph:
  requires: []
  provides:
    - dados/matches_train.parquet (8025 rows, 2003-2022)
    - dados/matches_test.parquet (1140 rows, 2023-2025)
    - notebook_data.ipynb (Phase 1 data pipeline)
  affects:
    - Phase 2 feature engineering (loads parquet)
    - Phase 3 baseline models (loads parquet, uses 49.6% naive baseline)
    - Phase 4 gradient boosting (loads parquet)
tech_stack:
  added:
    - seaborn 0.13.2
    - pytest 9.0.3
    - nbmake 1.5.5
  patterns:
    - pandas temporal split (year cutoff, no shuffle)
    - pyarrow parquet serialization
    - nbmake notebook-as-test via pytest
    - headless matplotlib via FigureCanvasAgg + IPython.display.Image
key_files:
  created:
    - notebook_data.ipynb
    - dados/matches_train.parquet
    - dados/matches_test.parquet
    - pytest.ini
  modified:
    - pyproject.toml (added seaborn, pytest, nbmake)
    - uv.lock (regenerated)
    - .planning/phases/01-data-ingestion-cleaning/01-VALIDATION.md (approved)
decisions:
  - "D-01: notebook_data.ipynb is the Phase 1 deliverable (dedicated data notebook)"
  - "D-02: Two parquet files at dados/matches_train.parquet and dados/matches_test.parquet"
  - "D-03: Year cutoff 2023; train <= 2022 (8025 rows), test >= 2023 (1140 rows)"
  - "D-04: result column with string labels HomeWin/Draw/AwayWin derived from vencedor"
  - "D-05: canonical_names dict applied before groupby; currently empty (no active variants)"
  - "HomeWin rate is 49.6%, NOT 46% — corrected discrepancy from CLAUDE.md/STATE.md"
metrics:
  duration: ~10 minutes
  completed_date: "2026-05-22"
  tasks_completed: 3
  files_created: 4
  files_modified: 3
---

# Phase 1 Plan 1: Data Ingestion & Cleaning Summary

**One-liner:** Pandas CSV-to-parquet pipeline with temporal split and 9-cell notebook producing 8025-row train / 1140-row test parquet files, locked by nbmake test runner running in 2.2s.

---

## What Was Built

### Files Created

| File | Description | Key Invariant |
|------|-------------|---------------|
| `notebook_data.ipynb` | 9-cell data pipeline notebook (executed, outputs embedded) | All 9165 rows processed; 8025/1140 split; 0 nulls |
| `dados/matches_train.parquet` | Cleaned training set 2003-2022 | 8025 rows, 11 columns, result ∈ {HomeWin,Draw,AwayWin} |
| `dados/matches_test.parquet` | Cleaned held-out test set 2023-2025 | 1140 rows, same schema, no date overlap with train |
| `pytest.ini` | nbmake test runner config | addopts = --nbmake -x; norecursedirs = .venv .git |

### Files Modified

| File | Change |
|------|--------|
| `pyproject.toml` | Added seaborn>=0.13.2, pytest>=9.0.3, nbmake>=1.5.5 |
| `uv.lock` | Regenerated after uv add |
| `.planning/phases/01-data-ingestion-cleaning/01-VALIDATION.md` | Updated from template to approved state with filled Per-Task Verification Map |

---

## Verified Data Facts

| Metric | Value | Source |
|--------|-------|--------|
| Total rows | 9,165 | campeonato-brasileiro-full.csv |
| Train rows (2003-2022) | 8,025 | Year cutoff <= 2022 |
| Test rows (2023-2025) | 1,140 | Year cutoff >= 2023 |
| HomeWin rate | **49.6%** | Computed from actual data |
| Draw rate | 26.4% | Computed from actual data |
| AwayWin rate | 23.9% | Computed from actual data |
| Unique team names | 46 | No normalization needed |
| Top-10 clubs >= 380 rows | All pass | Flamengo, Corinthians, Sao Paulo, Fluminense, Santos, Internacional, Atletico-MG, Athletico-PR, Gremio, Palmeiras |

---

## Output Schema

```
Columns (11): ID, date, season, round, home_team, away_team,
              home_score, away_score, home_state, away_state, result
Dtypes:
  ID:         int64
  date:       datetime64[ns]
  season:     int64 (year extracted from date)
  round:      int64
  home_team:  object
  away_team:  object
  home_score: int64
  away_score: int64
  home_state: object
  away_state: object
  result:     object ('HomeWin', 'Draw', 'AwayWin')
```

---

## Canonical-Names Dict State

The `canonical_names` dict in Cell 6 of `notebook_data.ipynb` is currently **empty** — no active variants exist in the 2003-2025 dataset (46 unique names, 0 multi-variant collisions verified by direct inspection). The dict acts as a defensive guard for future data additions. Populate it if new naming variants appear (e.g., `'Atletico MG': 'Atletico-MG'`).

---

## Naive Baseline Correction

**CORRECTION:** The naive "always HomeWin" baseline for this dataset is **49.6%**, NOT ~46% as stated in `CLAUDE.md` and `STATE.md`.

- The 46% figure appears to come from a global football average or a different dataset.
- Direct inspection of `campeonato-brasileiro-full.csv` confirms 49.6%.
- All Phase 3 baseline comparisons must use **49.6%**, not 46%.
- A Portuguese comment in the notebook's EDA cell documents this discrepancy.

---

## pytest --nbmake Runtime

`pytest --nbmake notebook_data.ipynb -x` runs in **~2.2 seconds** (measured: 2.16s and 2.21s across two runs on 2026-05-22 hardware).

---

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] EDA cell plt.show() did not embed PNG in notebook output**
- **Found during:** Task 3 execution
- **Issue:** `sns.countplot` + `plt.show()` with `MPLBACKEND=agg` produces stream output but not an embedded `display_data/image/png` in the notebook cell output.
- **Fix:** Used the established headless render pattern from `PATTERNS.md` (FigureCanvasAgg + BytesIO + `IPython.display.Image`). Replaced seaborn `countplot` with equivalent `matplotlib.figure.Figure` bar chart. The `display_data` with `image/png` key is now present in the executed cell output.
- **Files modified:** `notebook_data.ipynb` (EDA cell only)
- **Commit:** a5e9d75

**2. pyarrow already in pyproject.toml**
- **Found during:** Task 1
- **Observation:** `pyarrow>=24.0.0` was already present in `pyproject.toml` and installed in `.venv`. Only `seaborn`, `pytest`, and `nbmake` needed to be added.
- **Action:** Ran `uv add seaborn pytest nbmake` (not `uv add pyarrow seaborn pytest nbmake`).
- **Impact:** None — the outcome matches the plan requirement.

---

## Threat Flags

None. Phase 1 handles only local CSV files and writes local parquet files. No network calls, no authentication, no user input, no secrets.

---

## Self-Check: PASSED

- [x] `notebook_data.ipynb` exists: FOUND
- [x] `dados/matches_train.parquet` exists: FOUND (8025 rows)
- [x] `dados/matches_test.parquet` exists: FOUND (1140 rows)
- [x] `pytest.ini` exists: FOUND
- [x] Commit 749a7f1 exists (Task 1): FOUND
- [x] Commit 29123c6 exists (Task 2): FOUND
- [x] Commit a5e9d75 exists (Task 3): FOUND
- [x] `pytest --nbmake notebook_data.ipynb -x` exits 0: VERIFIED
- [x] HomeWin rate 49.6% assertion passes: VERIFIED
- [x] All 10 top clubs >= 380 rows: VERIFIED
- [x] No KFold/StratifiedKFold/shuffle=True/random_state in notebook: VERIFIED (count=0)
- [x] 01-VALIDATION.md has nyquist_compliant: true and wave_0_complete: true: VERIFIED
