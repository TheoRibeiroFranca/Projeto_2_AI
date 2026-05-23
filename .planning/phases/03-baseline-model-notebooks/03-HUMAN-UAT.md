---
status: resolved
phase: 03-baseline-model-notebooks
source: [03-VERIFICATION.md]
started: 2026-05-23T19:35:00Z
updated: 2026-05-23T19:55:00Z
---

## Current Test

[all items resolved]

## Tests

### 1. classification_report row-label correctness (CR-01)
expected: Per-class row labels in classification_report output must match the real HomeWin/Draw/AwayWin metrics.
result: passed — fixed inline during phase close (commit 2f937dc). Both notebooks now pass `labels=['HomeWin','Draw','AwayWin']` alongside `target_names`. Empirical re-run shows HomeWin (support=549), Draw (support=298), AwayWin (support=293) — matches real class distribution.

### 2. predict_match UserWarning noise (WR-01)
expected: Decide whether the benign sklearn UserWarning on predict_match is acceptable or should be patched.
result: passed — user-accepted as-is during phase close. The warning is cosmetic; predict_match still returns the correct label. Can be patched in Phase 4 alongside the same pattern in notebook_gradient_boost.ipynb if desired.

## Summary

total: 2
passed: 2
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps
