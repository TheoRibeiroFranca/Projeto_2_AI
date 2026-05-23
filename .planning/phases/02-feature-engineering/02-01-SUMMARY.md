---
phase: 02-feature-engineering
plan: "01"
subsystem: feature-engineering
tags: [pandas, rolling-window, feature-matrix, parquet, temporal-leakage-prevention]
dependency_graph:
  requires:
    - "01-01-SUMMARY.md (dados/matches_train.parquet, dados/matches_test.parquet)"
  provides:
    - "dados/feature_matrix_train.parquet (8025x31)"
    - "dados/feature_matrix_test.parquet (1140x31)"
    - "build_features(df) function in notebook_data.ipynb"
  affects:
    - "Phase 3/4 model notebooks (consume feature_matrix_*.parquet)"
tech_stack:
  added:
    - "scikit-learn==1.8.0 (installed via uv sync; Phase 3 prerequisite)"
    - "scipy==1.17.1, joblib==1.5.3, threadpoolctl==3.6.0 (scikit-learn deps)"
  patterns:
    - "season-scoped rolling: groupby(['team','season']).transform(lambda x: x.shift(1).rolling(5, min_periods=5))"
    - "D-06 imputation: prev-season last-10-match mean prior scaled by window size"
    - "D-07 fallback: 0.0 for teams with no prior season history"
    - "long-format team-match table pivot for match-level feature join"
key_files:
  created:
    - "dados/feature_matrix_train.parquet (8025 rows x 31 columns)"
    - "dados/feature_matrix_test.parquet (1140 rows x 31 columns)"
  modified:
    - "notebook_data.ipynb (3 cells added at indices 19, 20, 21)"
    - "uv.lock (scikit-learn + deps added)"
decisions:
  - "Omitted Cell D (## 11. Verificação de vazamento markdown placeholder) — Plan 02 will add both the markdown header and leakage assertion code cell together, keeping those two cells cohesive"
  - "Verification check for Phase 1 parquet overwrite applied only to Phase 2 cells (indices 19+), not all-cells-combined — the all-combined check from the plan had a false-positive because Phase 1 cell 16 legitimately writes to matches_train.parquet"
metrics:
  duration: "~30 minutes total execution"
  completed_date: "2026-05-23"
  tasks_completed: 2
  files_created: 2
  files_modified: 2
---

# Phase 2 Plan 01: Feature Engineering Pipeline Summary

Phase 2 feature engineering pipeline: `build_features(df)` rolling-window function producing 31-column feature matrix with zero NaN via D-06/D-07 imputation.

## What Was Built

Appended three cells to `notebook_data.ipynb` (after the existing 19 Phase 1 cells) to implement the complete Phase 2 feature engineering pipeline as a reusable `build_features(df)` function. Two new parquet files serve as the Phase 2 -> Phase 3/4 contract.

### Task 1: uv sync (Phase 3 prerequisite)

- Commit: `11ab680`
- Ran `uv sync` to install `scikit-learn==1.8.0` plus deps (`scipy==1.17.1`, `joblib==1.5.3`, `threadpoolctl==3.6.0`)
- `pyproject.toml` unchanged (dependency was already declared as `scikit-learn>=1.3`)
- pandas 2.3.3 and pyarrow 24.0.0 confirmed importable after sync

### Task 2: build_features function + apply/split/save cells

- Commit: `222f7df`
- Added 3 cells at notebook indices 19, 20, 21

**Cell 19 (markdown):** `## 10. Engenharia de features` — Portuguese section header explaining rolling windows and anti-leakage design.

**Cell 20 (code):** `build_features(df)` function — pure function implementing all four FEAT requirements:
- FEAT-01: `goals_scored/conceded_last5` via season-scoped `.shift(1).rolling(5, min_periods=5).mean()`
- FEAT-02: `win/drw/lss_last5` via same pattern with `.sum()`
- FEAT-03: `home_win/draw/loss_pct_season`, `away_win/draw/loss_pct_season` via `.shift(1).expanding()` on separate home/away venue sub-tables
- FEAT-04: `goal_diff_last5` and `points_last5` derived after imputation (arithmetic on FEAT-01/02 columns)
- D-06 imputation: previous-season last-10-match mean (scale=1.0 for means, scale=5.0 for sums)
- D-07 fallback: `.fillna(0.0)` for teams with no prior season history

**Cell 21 (code):** Apply + split + save — reads Phase 1 parquets, calls `build_features(combined)`, re-splits by `season` column, saves both feature matrix parquets with round-trip assertions.

## Files Created/Modified

| File | Change | Details |
|------|--------|---------|
| `dados/feature_matrix_train.parquet` | Created | 8025 rows x 31 columns, zero NaN in feature columns |
| `dados/feature_matrix_test.parquet` | Created | 1140 rows x 31 columns, zero NaN in feature columns |
| `notebook_data.ipynb` | Modified | 3 cells appended at indices 19, 20, 21 |
| `uv.lock` | Modified | scikit-learn 1.8.0 + scipy/joblib/threadpoolctl added |

## Phase 1 Parquets: Unchanged

Phase 1 parquets `dados/matches_train.parquet` and `dados/matches_test.parquet` were NOT modified by Phase 2 cells. Verified by round-trip assertion in Cell 21:
- `dados/matches_train.parquet`: 8025 rows x 11 columns (unchanged schema)
- `dados/matches_test.parquet`: 1140 rows x 11 columns (unchanged schema)

## 20 Feature Columns and NaN Counts

All 20 feature columns have **0 NaN** in both train and test after D-06/D-07 imputation.

| Column | FEAT | NaN (train) | NaN (test) |
|--------|------|-------------|------------|
| `home_goals_scored_last5` | 01 | 0 | 0 |
| `home_goals_conceded_last5` | 01 | 0 | 0 |
| `away_goals_scored_last5` | 01 | 0 | 0 |
| `away_goals_conceded_last5` | 01 | 0 | 0 |
| `home_wins_last5` | 02 | 0 | 0 |
| `home_draws_last5` | 02 | 0 | 0 |
| `home_losses_last5` | 02 | 0 | 0 |
| `away_wins_last5` | 02 | 0 | 0 |
| `away_draws_last5` | 02 | 0 | 0 |
| `away_losses_last5` | 02 | 0 | 0 |
| `home_win_pct_season` | 03 | 0 | 0 |
| `home_draw_pct_season` | 03 | 0 | 0 |
| `home_loss_pct_season` | 03 | 0 | 0 |
| `away_win_pct_season` | 03 | 0 | 0 |
| `away_draw_pct_season` | 03 | 0 | 0 |
| `away_loss_pct_season` | 03 | 0 | 0 |
| `home_goal_diff_last5` | 04 | 0 | 0 |
| `home_points_last5` | 04 | 0 | 0 |
| `away_goal_diff_last5` | 04 | 0 | 0 |
| `away_points_last5` | 04 | 0 | 0 |

RESEARCH.md notes ~12.9% NaN rate in long-format before imputation. Post-imputation NaN rate = 0% in both feature matrices.

## build_features Function Contract

- **Defined in:** `notebook_data.ipynb` cell index 20 (single code cell)
- **Signature:** `def build_features(df: pd.DataFrame) -> pd.DataFrame`
- **Input:** DataFrame matching Phase 1 parquet schema (any filtered date range)
- **Output:** DataFrame with `len(input)` rows and 31 columns in the exact contract order
- **Side effects:** None (pure function — no print statements, no file I/O, no global mutations)

## NaN Rate Before vs After Imputation

- **Pre-imputation (long-format FEAT-01/02):** ~12.9% NaN rate (RESEARCH.md verified figure; 474/474 first-match-per-team-season rows are NaN confirming correct `.shift(1)` application)
- **Post-imputation (match-level feature columns):** 0 NaN in all 20 feature columns in both train and test parquets

## pytest --nbmake Runtime

`pytest --nbmake notebook_data.ipynb -x` completed in **4.25 seconds** (real time). This is well within the VALIDATION.md estimated runtime.

## Cell D Markdown Placeholder Decision

**Choice made: Cell D (## 11. Verificação de vazamento placeholder) omitted.** Plan 02 will add both the `## 11. Verificação de vazamento` markdown header AND the leakage assertion code cell together in a single cohesive block. Adding only the markdown placeholder now without the code cell would leave a section header with no content, which is worse for notebook readability than having Plan 02 add them as a pair.

## Deviations from Plan

### Plan Deviation 1: Verification script false-positive on Phase 1 parquet check

**Found during:** Task 2 verification
**Issue:** The plan's `<automated>` verification block checks ALL notebook code source combined for `to_parquet('dados/matches_train.parquet'` absence. This produces a false positive because Phase 1 Cell 16 legitimately writes to `matches_train.parquet` — this is the correct behavior, not an error.
**Fix:** Applied the check only to Phase 2 cells (notebook indices 19+), which is the correct semantics of "Phase 2 cells must not overwrite Phase 1 outputs."
**Result:** Verification passes correctly; Phase 1 cells unaffected.

### Plan Deviation 2: Task commits went to main branch, worktree branch reset to reconcile

**Found during:** SUMMARY.md commit
**Issue:** Task 1 and Task 2 commits were made while running git in `/home/theo/AI/Projeto_2_AI` (main repo) rather than the worktree directory. Both commits landed on main branch instead of the worktree branch.
**Fix:** Used `git reset --hard 222f7df` to advance the worktree branch to main's tip (which includes all task work), then recommitted SUMMARY.md on top of the complete work history.
**Result:** Worktree branch correctly includes all three commits: uv sync, build_features, and SUMMARY.md.

## Threat Flags

None — this plan is a pure local data transformation pipeline with no network calls, no authentication paths, no user input, and no new file access patterns beyond the existing `dados/` directory.

## Known Stubs

None — `build_features(df)` is fully implemented and wired. Both feature matrix parquets are materialized on disk with correct shapes and zero NaN.

## Self-Check: PASSED

Files exist:
- `dados/feature_matrix_train.parquet` — FOUND
- `dados/feature_matrix_test.parquet` — FOUND
- `.planning/phases/02-feature-engineering/02-01-SUMMARY.md` — this file

Commits exist:
- `11ab680` — FOUND (chore: uv sync)
- `222f7df` — FOUND (feat: build_features + feature matrices)
