---
status: partial
phase: 03-baseline-model-notebooks
source: [03-VERIFICATION.md]
started: 2026-05-23T19:35:00Z
updated: 2026-05-23T19:35:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. classification_report row-label correctness (CR-01)
expected: Reader must understand whether the printed `HomeWin`/`AwayWin` rows correspond to the real HomeWin/AwayWin class metrics. CR-01 from 03-REVIEW.md is empirically confirmed — sklearn sorts labels alphabetically (`AwayWin`, `Draw`, `HomeWin`) and applies `target_names` positionally, so the printed `HomeWin` row actually contains AwayWin's metrics (support=293) and the printed `AwayWin` row actually contains HomeWin's metrics (support=549). The `Draw` row is correct only by alphabetical coincidence. Macro-F1 and accuracy aggregates are unaffected; the confusion-matrix heatmap (Cell 13) uses explicit `labels=['HomeWin','Draw','AwayWin']` and is correct.
result: [pending]

### 2. predict_match UserWarning noise (WR-01)
expected: Calling `predict_match('Flamengo', 'Palmeiras')` triggers `UserWarning: X does not have valid feature names, but {Model} was fitted with feature names`. The warning is benign — the notebook still produces a valid W/D/L label and the prediction is correct. Decide whether the noisy output is acceptable for submission or whether WR-01 should be patched (wrap vector in a `pd.DataFrame` with FEATURE_COLS columns before `.predict()`).
result: [pending]

## Summary

total: 2
passed: 0
issues: 0
pending: 2
skipped: 0
blocked: 0

## Gaps
