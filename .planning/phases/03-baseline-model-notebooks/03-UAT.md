---
status: complete
phase: 03-baseline-model-notebooks
source: [03-01-SUMMARY.md, 03-02-SUMMARY.md]
started: 2026-05-23T20:00:00Z
updated: 2026-05-23T20:10:00Z
---

## Current Test

[testing complete]

## Tests

### 1. Logistic Regression notebook runs end-to-end
expected: Run `pytest --nbmake notebook_logistic.ipynb -x`. Expected: 1 passed, 0 errors, exit code 0. All 18 cells execute without exceptions.
result: pass

### 2. Random Forest notebook runs end-to-end
expected: Run `pytest --nbmake notebook_random_forest.ipynb -x`. Expected: 1 passed, 0 errors, exit code 0. All 18 cells execute without exceptions.
result: pass

### 3. Logistic Regression classification report shows all 3 classes
expected: Row labels must be HomeWin, Draw, AwayWin (not 0, 1, 2). Draw recall must be above 0.05.
result: pass

### 4. Random Forest classification report shows all 3 classes with non-zero Draw recall
expected: Row labels must be HomeWin, Draw, AwayWin. Draw recall must be > 0.05 (expected ~0.19 with min_samples_leaf=5).
result: pass

### 5. Naive baseline comparison is dynamic (not hardcoded)
expected: Naive accuracy computed from `(y_test == 'HomeWin').mean()` — no hardcoded value like 0.496 or 0.46.
result: pass

### 6. predict_match returns valid label in Logistic Regression notebook
expected: `predict_match('Flamengo', 'Palmeiras')` returns one of: HomeWin, Draw, or AwayWin — no execution error.
result: pass

### 7. predict_match returns valid label in Random Forest notebook
expected: `predict_match('Flamengo', 'Palmeiras')` returns one of: HomeWin, Draw, or AwayWin — no execution error.
result: pass

## Summary

total: 7
passed: 7
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps

[none]
