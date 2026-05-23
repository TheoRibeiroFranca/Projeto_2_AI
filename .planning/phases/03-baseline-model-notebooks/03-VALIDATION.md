---
phase: 3
phase-slug: baseline-model-notebooks
date: 2026-05-23
status: active
---

# Phase 3 Validation Strategy

## Test Framework

| Property | Value |
|----------|-------|
| Framework | pytest + nbmake |
| Config file | pyproject.toml (existing, from Phase 1) |
| Quick run command | `pytest --nbmake notebook_logistic.ipynb -x` |
| Full suite command | `pytest --nbmake notebook_logistic.ipynb notebook_random_forest.ipynb -x` |

## Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| MODEL-01 | notebook_logistic.ipynb runs end-to-end | smoke | `pytest --nbmake notebook_logistic.ipynb -x` | No — Wave 0 |
| MODEL-02 | notebook_random_forest.ipynb runs end-to-end | smoke | `pytest --nbmake notebook_random_forest.ipynb -x` | No — Wave 0 |
| MODEL-04 | TimeSeriesSplit used (no KFold) | smoke (cell execution) | `pytest --nbmake ... -x` — cell errors if TSS not imported | No — Wave 0 |
| EVAL-01 | predict_match('Flamengo', 'Palmeiras') returns valid W/D/L label | smoke (example cell) | Notebook last cell calls predict_match, nbmake verifies no exception | No — Wave 0 |
| EVAL-02 | classification_report printed without error | smoke | Notebook eval cell, verified by nbmake | No — Wave 0 |
| EVAL-03 | sns.heatmap renders without error | smoke | Notebook heatmap cell, verified by nbmake | No — Wave 0 |

## Sampling Rate

- **Per task commit:** `pytest --nbmake notebook_logistic.ipynb -x` (or RF equivalent)
- **Per wave merge:** `pytest --nbmake notebook_logistic.ipynb notebook_random_forest.ipynb -x`
- **Phase gate:** Both notebooks pass nbmake before `/gsd:verify-work`

## Wave 0 Gaps (to be created by this phase)

- [ ] `notebook_logistic.ipynb` — covers MODEL-01, MODEL-04, EVAL-01, EVAL-02, EVAL-03
- [ ] `notebook_random_forest.ipynb` — covers MODEL-02, MODEL-04, EVAL-01, EVAL-02, EVAL-03

*Both notebooks are the deliverables of this phase. They are test artifacts AND implementation artifacts simultaneously — nbmake tests run the notebooks end-to-end.*

## Critical Validation Notes

- **Draw recall gate:** Non-zero Draw recall is the hardest requirement. RF must use `min_samples_leaf=5`; LR relies on `class_weight='balanced'`. If Draw recall = 0, the notebook FAILS verification.
- **No hardcoded baselines:** Naive baseline is computed dynamically via `(y_test == 'HomeWin').mean()` — never hardcoded as 0.46 or 0.496.
- **Temporal integrity already guaranteed:** Feature matrices loaded from parquet are NaN-free and temporally clean (Phase 2 output). Model notebooks must not re-run feature engineering.
