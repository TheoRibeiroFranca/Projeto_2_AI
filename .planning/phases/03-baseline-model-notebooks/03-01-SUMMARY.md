---
phase: 03-baseline-model-notebooks
plan: 01
subsystem: ml-modeling
tags: [scikit-learn, logistic-regression, time-series-split, classification-report, seaborn-heatmap, predict-match, class-balanced]

# Dependency graph
requires:
  - phase: 02-feature-engineering
    provides: dados/feature_matrix_train.parquet and dados/feature_matrix_test.parquet (20 numeric features + 11 base cols, NaN-free)
provides:
  - notebook_logistic.ipynb — self-contained Logistic Regression baseline with TSS CV, classification_report, seaborn confusion matrix, dynamic naive baseline comparison, and predict_match(home_team, away_team) lookup function
affects: [03-02-random-forest, 04-gradient-boost, 04-documentation-polish]

# Tech tracking
tech-stack:
  added: []  # all packages already present in venv (scikit-learn 1.8.0, pandas 2.3.3, seaborn 0.13.2, pyarrow 24.0.0)
  patterns:
    - "MPLBACKEND=agg + MPLCONFIGDIR set via os.environ before any matplotlib/seaborn import (Pitfall 5)"
    - "TimeSeriesSplit(n_splits=5) passed as cv= to cross_val_score — no KFold/StratifiedKFold (D-04)"
    - "class_weight='balanced' on first fit() call to enforce non-zero Draw/AwayWin recall (D-02)"
    - "predict_match lookup: pd.concat(train, test).sort_values('date'), groupby(home_team).last() / groupby(away_team).last(), reindex via [FEATURE_COLS] before reshape (Pitfall 4)"
    - "Naive baseline computed dynamically via (y_test == 'HomeWin').mean() — never hardcoded (Pitfall 3, D-09)"
    - "ValueError with team name + VALID_TEAMS suggestion when team is unknown (D-06)"

key-files:
  created:
    - "notebook_logistic.ipynb"
  modified: []

key-decisions:
  - "Used lbfgs solver with max_iter=1000 (research confirmed identical results vs saga, converges in <1000 iter)"
  - "TimeSeriesSplit(n_splits=5) — sklearn default, ~1337-row validation folds, statistically sufficient"
  - "All 20 feature columns selected by exclusion of 11 NON_FEATURE_COLS — no hand-picking"
  - "predict_match reindex via pd.concat([home_row, away_row])[FEATURE_COLS] guards against column-order drift between home_last / away_last and training column order"

patterns-established:
  - "Section-divider markdown cells only (D-10) — no prose explanations in Phase 3 notebooks"
  - "Lookup-based predict_match using combined train+test parquet (D-05) — does not call build_features() from notebook_data.ipynb"
  - "Report macro-F1 alongside accuracy because class_weight='balanced' trades raw accuracy for minority-class recall"

requirements-completed: [MODEL-01, MODEL-04, EVAL-01, EVAL-02, EVAL-03]

# Metrics
duration: 3min
completed: 2026-05-23
---

# Phase 3 Plan 01: Logistic Regression Baseline Notebook Summary

**Self-contained LogisticRegression notebook with TimeSeriesSplit CV, class-balanced training, seaborn confusion matrix, and lookup-based predict_match function — passes nbmake end-to-end.**

## Performance

- **Duration:** ~3 min
- **Started:** 2026-05-23T17:57:02Z
- **Completed:** 2026-05-23T17:59:46Z
- **Tasks:** 2
- **Files modified:** 1 (created)

## Accomplishments

- `notebook_logistic.ipynb` created with 18 cells covering setup, data loading, feature selection, model training, TSS cross-validation, classification report, confusion matrix heatmap, naive baseline comparison, and predict_match lookup.
- `pytest --nbmake notebook_logistic.ipynb -x` exits 0 — full end-to-end execution verified.
- Empirical results match RESEARCH.md predictions exactly: test accuracy 0.4404, macro-F1 0.4120, Draw recall 0.25 (well above the 0.05 threshold), naive baseline 0.4816 (computed dynamically — no hardcoded value).
- `predict_match('Flamengo', 'Palmeiras')` returns a valid W/D/L label (`HomeWin`); unknown-team input raises `ValueError` containing the team name and the full VALID_TEAMS list.
- TimeSeriesSplit(n_splits=5) CV reports accuracy 0.392 ± 0.020 and macro-F1 0.365 ± 0.016 — no KFold/StratifiedKFold anywhere in the source.

## Task Commits

Each task was committed atomically:

1. **Task 1: Create notebook cells 1-15 (setup through baseline comparison)** — `1361a30` (feat)
2. **Task 2: Add predict_match cells 16-18 and validate via nbmake** — `cdc7fe5` (feat)

## Files Created/Modified

- `notebook_logistic.ipynb` — 18-cell self-contained Logistic Regression baseline notebook. Loads `dados/feature_matrix_{train,test}.parquet`, trains `LogisticRegression(class_weight='balanced', solver='lbfgs', max_iter=1000, random_state=42)`, reports TimeSeriesSplit CV scores (accuracy + macro-F1), prints classification_report per class, renders seaborn confusion matrix heatmap, compares against the dynamically-computed naive baseline, and exposes `predict_match(home_team, away_team)` returning W/D/L (with ValueError on unknown teams).

## Decisions Made

- **lbfgs over saga:** Research confirmed identical results; lbfgs converges in <1000 iterations on this dataset.
- **TimeSeriesSplit n_splits=5:** sklearn default, gives 1337-row validation folds — statistically sufficient. No GridSearchCV per D-03.
- **predict_match reindex via `[FEATURE_COLS]` after concat:** Explicitly prevents Pitfall 4 (column-order drift between `home_last`/`away_last` Series and the original training column order).
- **Macro-F1 framed as primary metric:** D-09 requires comparison against the 49.6% (project-wide) / 48.16% (test-set) naive baseline; with `class_weight='balanced'` the model trades raw accuracy for minority-class recall, so the baseline comparison cell prints both accuracy and macro-F1 and explicitly states this tradeoff.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- A benign sklearn UserWarning ("X does not have valid feature names, but LogisticRegression was fitted with feature names") is emitted by cell 18 because `predict_match` passes a numpy array via `.values.reshape(1, -1)` to a model fit on a DataFrame. nbmake still exits 0 — the warning is not a failure. Left as-is since the plan does not require warning-free output and fixing it (wrapping the array in a DataFrame) would add a stylistic cell modification with no behavioral change.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- `notebook_logistic.ipynb` is the LR baseline ready to be referenced from the RF notebook (03-02) for comparison.
- The `predict_match` lookup pattern (combined parquet → groupby last → reindex via FEATURE_COLS) is fully validated and ready to be reused verbatim in `notebook_random_forest.ipynb` (swap `lr` for `rf`).
- TSS CV scoring pattern (`cross_val_score(..., cv=TimeSeriesSplit(n_splits=5), scoring=...)`) ready to reuse for RF.
- Pattern for dynamic naive baseline (`(y_test == 'HomeWin').mean()`) ready to reuse — eliminates Pitfall 3 risk for the RF notebook.
- 03-02 (Random Forest) can proceed immediately; both plans are in Wave 1 and have no inter-plan dependency.

## Self-Check

- `[ -f notebook_logistic.ipynb ]` → FOUND
- Commit `1361a30` → FOUND in git log
- Commit `cdc7fe5` → FOUND in git log
- nbmake exit code: 0 (1 passed)

## Self-Check: PASSED

---
*Phase: 03-baseline-model-notebooks*
*Completed: 2026-05-23*
