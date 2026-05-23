---
phase: 03-baseline-model-notebooks
reviewed: 2026-05-23T00:00:00Z
depth: standard
files_reviewed: 2
files_reviewed_list:
  - notebook_logistic.ipynb
  - notebook_random_forest.ipynb
findings:
  critical: 1
  warning: 2
  info: 3
  total: 6
status: issues_found
---

# Phase 3: Code Review Report

**Reviewed:** 2026-05-23
**Depth:** standard
**Files Reviewed:** 2
**Status:** issues_found

## Summary

Both notebooks satisfy the surface-level acceptance criteria from `03-01-PLAN.md` and `03-02-PLAN.md`: they load the correct feature parquets, fit class-balanced models, use `TimeSeriesSplit` exclusively, dynamically compute the naive baseline, expose `predict_match()` with `ValueError` on unknown teams, and avoid the forbidden `KFold` / `StratifiedKFold` / `shuffle=True` / hard-coded `0.46` / `0.496` patterns. The mandatory project gotchas (class_weight balanced, `min_samples_leaf=5` on RF, `MPLBACKEND=agg` before imports, no `matches_*.parquet`, no re-derivation of `result`) are all respected.

However, the central evaluation output — `classification_report` — is **mislabeled in both notebooks**, producing a report whose first row is annotated `HomeWin` but actually carries `AwayWin`'s metrics (and vice versa). Anyone reading the notebook will draw incorrect conclusions about class imbalance, recall, and naive-baseline framing. This is a correctness defect, not a style issue, and it propagates into the phase verification step ("Draw recall non-zero") only by alphabetical coincidence. Two warning-level issues (sklearn feature-name UserWarning leaked into predict, partial-match ValueError) and three info-level items round out the review.

## Critical Issues

### CR-01: `classification_report` rows are mislabeled in both notebooks — HomeWin and AwayWin metrics are swapped

**Files:**
- `notebook_logistic.ipynb` — Cell 12 (eval cell, id `31a13415`)
- `notebook_random_forest.ipynb` — Cell 12 (eval cell, id `878b245d`)

**Issue:**
Both notebooks call:
```python
classification_report(y_test, y_pred, target_names=['HomeWin', 'Draw', 'AwayWin'])
```

`sklearn.metrics.classification_report` does **not** reorder rows to match `target_names`; it sorts the actual labels alphabetically and then applies `target_names` positionally. With string labels `'HomeWin' / 'Draw' / 'AwayWin'`, sklearn's internal order is `['AwayWin', 'Draw', 'HomeWin']`. Passing `target_names=['HomeWin', 'Draw', 'AwayWin']` therefore renames:
- AwayWin's row → printed as `HomeWin`
- Draw's row → printed as `Draw` (correct only by alphabetical coincidence)
- HomeWin's row → printed as `AwayWin`

Verified empirically against the actual test parquet — the printed `HomeWin` row shows `support=293` and the printed `AwayWin` row shows `support=549`. The real distribution is `HomeWin=549, Draw=298, AwayWin=293`. Every reader of the notebook will conclude that AwayWin is the majority class and HomeWin the minority class — the inverse of reality. The same swap appears in cross-validation framing, naive-baseline framing, and the project's draw-class-collapse diagnostic (the empirical numbers in RESEARCH.md were derived from a correctly-labeled run, so the discrepancy is silent).

The acceptance criterion "Draw recall in the classification_report is non-zero (>0.05)" passes by accident — Draw happens to land in alphabetical slot 1, the same slot the planner intended for it. Any reader who interprets the HomeWin/AwayWin rows for downstream decisions (Phase 4 ensemble weighting, error analysis, model comparison) inherits the swap.

**Fix:**
Pass the labels in alphabetical order so they match sklearn's internal sort, or — preferred — pass `labels=` explicitly to lock the order, then keep `target_names=` aligned:

```python
# Option A (preferred): explicit labels, matched target_names
print(classification_report(
    y_test, y_pred,
    labels=['HomeWin', 'Draw', 'AwayWin'],
    target_names=['HomeWin', 'Draw', 'AwayWin'],
))

# Option B: drop target_names; sklearn will print the real labels alphabetically
print(classification_report(y_test, y_pred))
```

Apply the same fix in both notebooks. After the fix, the printed `HomeWin` row should show `support=549` for the test set.

## Warnings

### WR-01: `predict_match` passes a raw `np.ndarray` to a model fitted on a `DataFrame`, emitting a sklearn UserWarning on every call

**Files:**
- `notebook_logistic.ipynb` — Cell 17 (predict_match definition, id `702a967b`), Cell 18 (example call, id `1e0272b0`)
- `notebook_random_forest.ipynb` — Cell 17 (predict_match definition, id `b838cc87`), Cell 18 (example call, id `2d03d47f`)

**Issue:**
Both notebooks fit on `X_train = train[FEATURE_COLS]` (a `DataFrame`, so sklearn records `feature_names_in_`), but `predict_match` does:
```python
vector = row.values.reshape(1, -1)
return lr.predict(vector)[0]   # or rf.predict
```
This passes a feature-name-less ndarray to a feature-name-aware estimator, triggering:
```
UserWarning: X does not have valid feature names, but LogisticRegression was fitted with feature names
```
on every `predict_match` call. The result is still correct (column ordering is fixed by the `[FEATURE_COLS]` reindex two lines earlier), but the notebook output is polluted with a warning that suggests a bug, and the example cell will display this warning to anyone running the notebook.

**Fix:**
Keep the feature vector as a `DataFrame` with matching column names:
```python
home_row = home_last.loc[home_team]
away_row = away_last.loc[away_team]
row = pd.concat([home_row, away_row])[FEATURE_COLS]
vector = pd.DataFrame([row.values], columns=FEATURE_COLS)
return lr.predict(vector)[0]   # or rf.predict
```

### WR-02: `predict_match` rejects unknown teams loudly but cannot detect partial matches or capitalization mistakes

**Files:**
- `notebook_logistic.ipynb` — Cell 17 (id `702a967b`)
- `notebook_random_forest.ipynb` — Cell 17 (id `b838cc87`)

**Issue:**
The membership checks (`if home_team not in home_last.index`) are exact-match string lookups. The dataset contains canonical names like `'Atletico-MG'`, `'Athletico-PR'`, `'Botafogo-RJ'`. A user typing `'Atletico MG'` (space instead of dash), `'flamengo'` (lowercase), or `'Sao Paulo '` (trailing whitespace) will hit the `ValueError` path even though the team is known. The `ValueError` does include the full `VALID_TEAMS` list, which helps, but the message dumps all 46 names inline — making it hard to spot the intended team.

This is also relevant because CLAUDE.md explicitly warns about team-name normalization variants ("Atletico-MG" vs "Atletico MG"). Phase 2 normalized the parquet, but `predict_match` is the user-facing entry point — any mismatch between user input and canonical form fails with no suggestion.

**Fix:**
At minimum, strip whitespace and surface near-matches using `difflib`:
```python
import difflib

def predict_match(home_team: str, away_team: str) -> str:
    home_team = home_team.strip()
    away_team = away_team.strip()
    for label, name, index in [('home_team', home_team, home_last.index),
                                ('away_team', away_team, away_last.index)]:
        if name not in index:
            suggestions = difflib.get_close_matches(name, VALID_TEAMS, n=3, cutoff=0.6)
            hint = f" Did you mean: {suggestions}?" if suggestions else ""
            raise ValueError(f"Unknown {label} '{name}'.{hint} Valid teams: {VALID_TEAMS}")
    ...
```

## Info

### IN-01: `np` import is unused in both notebooks

**Files:**
- `notebook_logistic.ipynb` — Cell 1 (id `23ffde95`), line `import numpy as np`
- `notebook_random_forest.ipynb` — Cell 1 (id `9fff1280`), line `import numpy as np`

**Issue:**
`numpy` is imported as `np` but never referenced — neither `np.something(...)` nor `np.<anything>` appears anywhere in either notebook. The plan documents mention "numpy used inline in predict_match implementation" (RESEARCH.md Standard Stack table), but the final implementation calls `.reshape(1, -1)` on a pandas-derived `.values` ndarray without going through `np.*`.

**Fix:**
Remove the unused import, or leave a comment explaining why it's kept. Low-risk cleanup.

### IN-02: Example `predict_match` call uses misspelled bait team name (`'TimeVinventado'`)

**Files:**
- `notebook_logistic.ipynb` — Cell 18 (id `1e0272b0`)
- `notebook_random_forest.ipynb` — Cell 18 (id `2d03d47f`)

**Issue:**
The negative-path example invokes `predict_match('TimeVinventado', 'Palmeiras')`. The intended Portuguese phrase is `Time Inventado` ("made-up team"). `TimeVinventado` is a typo (`V` doubled with `I`) that's harmless functionally — it still triggers the `ValueError` — but it's the only string in the notebook that's not English or canonical Portuguese.

**Fix:**
Rename to `'TimeInventado'` or simply `'NotARealTeam'` for clarity. Cosmetic.

### IN-03: Code duplication between the two notebooks is intentional but unbounded — a defect in one will only land in the other if a reviewer notices

**Files:**
- `notebook_logistic.ipynb` — all cells
- `notebook_random_forest.ipynb` — all cells

**Issue:**
The plan explicitly states "some code duplication between [notebooks] is acceptable" and forbids `.py` modules. As a result, `predict_match`, the lookup-table construction, the feature column list, and the evaluation cells are duplicated verbatim. The mis-labeled `classification_report` bug in CR-01 propagated to both files because of this duplication. There is no mechanism (besides this review) to keep them in sync. Future bug fixes will need to land in both notebooks manually.

**Fix:**
No action required for Phase 3 per the plan's no-`.py`-modules constraint. Phase 4 (the "polish" phase) could consolidate shared logic into a single shared notebook or accept the duplication explicitly. Flagging here so the duplication is documented as a known maintenance cost.

---

_Reviewed: 2026-05-23_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
