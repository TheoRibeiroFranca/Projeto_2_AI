---
phase: 04-advanced-model-notebook-polish
reviewed: 2026-05-23T00:00:00Z
depth: standard
files_reviewed: 3
files_reviewed_list:
  - notebook_gradient_boost.ipynb
  - notebook_logistic.ipynb
  - notebook_random_forest.ipynb
findings:
  critical: 2
  warning: 3
  info: 2
  total: 7
status: issues_found
---

# Phase 04: Code Review Report

**Reviewed:** 2026-05-23
**Depth:** standard
**Files Reviewed:** 3
**Status:** issues_found

## Summary

All three model notebooks (Logistic Regression, Random Forest, Gradient Boosting) share the same structural skeleton: load parquet, select features, fit, cross-validate with TimeSeriesSplit, evaluate, compare to naive baseline, expose a `predict_match` function. The temporal split is correct (no KFold/StratifiedKFold), `class_weight='balanced'` is present on all three models, and result encoding matches the contract (`HomeWin`/`Draw`/`AwayWin`).

Two critical issues were found: a feature-vector assembly bug in `predict_match` that silently produces a misaligned input vector under certain column orderings, and missing feature scaling in the Logistic Regression notebook that materially degrades model quality. Three warnings cover missing NaN guards before fit, a documented naive-baseline value that contradicts the CLAUDE.md contract, and an incomplete `VALID_TEAMS` set that can surface misleading error messages. Two info items cover minor dead-import and code-duplication notes.

---

## Critical Issues

### CR-01: `predict_match` — silent feature misalignment when `pd.concat` produces duplicate index labels

**File:** `notebook_gradient_boost.ipynb` (predict_match cell), `notebook_logistic.ipynb` (predict_match cell), `notebook_random_forest.ipynb` (predict_match cell)
**Issue:** The feature vector is assembled as:

```python
row = pd.concat([home_row, away_row])[FEATURE_COLS]
vector = pd.DataFrame([row.values], columns=FEATURE_COLS)
```

`pd.concat([home_row, away_row])` produces a Series whose index contains the original column names. The subsequent `[FEATURE_COLS]` label-based selection depends entirely on those labels being unique and present. If any feature column name does not start with `home_` or `away_` (e.g., a future feature named `goal_diff_abs`), it would be absent from both `home_row` and `away_row`, making `[FEATURE_COLS]` raise a `KeyError` — or worse, silently return `NaN` if pandas reindexes instead of raising. Additionally, `.values` is extracted *after* the label selection, but `pd.DataFrame([row.values], columns=FEATURE_COLS)` assumes `row` preserves the exact order of `FEATURE_COLS`. If `pd.concat` preserves insertion order (which it does in pandas >=1.0), this is currently safe — but only as long as all 20 features start with `home_` or `away_`. The assumption is fragile and undocumented.

More concretely: `row.values` is a numpy array whose position mapping to `FEATURE_COLS` relies on the implicit ordering guarantee of `pd.concat`. If the feature schema ever gains a column that doesn't start with `home_` or `away_`, the vector silently contains `NaN` at that position and the model predicts on corrupted input with no error raised.

**Fix:** Build the vector explicitly using the known column sets, and assert completeness:

```python
def predict_match(home_team: str, away_team: str) -> str:
    home_team = home_team.strip()
    away_team = away_team.strip()
    for label, name, index in [
        ('home_team', home_team, home_last.index),
        ('away_team', away_team, away_last.index),
    ]:
        if name not in index:
            suggestions = difflib.get_close_matches(name, VALID_TEAMS, n=3, cutoff=0.6)
            hint = f' Did you mean: {suggestions}?' if suggestions else ''
            raise ValueError(f"Unknown {label} '{name}'.{hint} Valid teams: {VALID_TEAMS}")

    home_row = home_last.loc[home_team]
    away_row = away_last.loc[away_team]

    # Build vector column-by-column in FEATURE_COLS order to avoid
    # any ordering assumptions from pd.concat
    combined_series = pd.concat([home_row, away_row])
    missing = [c for c in FEATURE_COLS if c not in combined_series.index]
    if missing:
        raise RuntimeError(f"Feature columns missing from lookup: {missing}")
    vector = pd.DataFrame([combined_series[FEATURE_COLS].values], columns=FEATURE_COLS)
    return model.predict(vector)[0]
```

This fix makes the ordering contract explicit and surfaces schema drift immediately instead of silently passing NaN to the estimator. Apply to all three notebooks.

---

### CR-02: Logistic Regression trained without feature scaling — results materially degraded

**File:** `notebook_logistic.ipynb` (Model Training cell)
**Issue:** `LogisticRegression` with `solver='lbfgs'` is a gradient-based optimizer that is sensitive to feature scale. The 20 feature columns span very different ranges:

- `home_wins_last5`, `home_draws_last5`, `home_losses_last5`: 0–5 (integer counts)
- `home_goals_scored_last5`, `home_goals_conceded_last5`: 0–~25 (sum over 5 games)
- `home_win_pct_season`, `home_draw_pct_season`, `home_loss_pct_season`: 0.0–1.0 (fractions)
- `home_points_last5`: 0–15

Without standardization, `lbfgs` takes many more iterations to converge and the resulting coefficients are numerically dominated by high-magnitude features (goals counts) regardless of their predictive power. The notebook output confirms underperformance: CV accuracy of 0.397 and macro-F1 of 0.367, while the Random Forest (unscaled) achieves 0.434 / 0.350. Logistic Regression should approach or exceed RF on a linearly separable signal when properly scaled; the gap here is consistent with poor conditioning.

`max_iter=1000` partially compensates but does not eliminate the conditioning problem. This means the published logistic regression baseline is pessimistic and not a fair measurement of what a linear model can achieve on this data.

**Fix:** Add a `StandardScaler` pipeline before the classifier:

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

lr_pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('clf', LogisticRegression(
        class_weight='balanced',
        solver='lbfgs',
        max_iter=1000,
        random_state=42,
    )),
])
lr_pipe.fit(X_train, y_train)

# Update cross_val_score and predict to use lr_pipe instead of lr
cv_acc = cross_val_score(lr_pipe, X_train, y_train, cv=tss, scoring='accuracy')
y_pred = lr_pipe.predict(X_test)
```

In `predict_match`, replace `lr.predict(vector)[0]` with `lr_pipe.predict(vector)[0]`. The scaler is inside the pipeline so it applies automatically to the 1-row prediction vector without any additional changes.

---

## Warnings

### WR-01: No NaN guard before `.fit()` — upstream schema change would cause silent mismatch

**File:** `notebook_logistic.ipynb` (Model Training cell), `notebook_random_forest.ipynb` (Model Training cell), `notebook_gradient_boost.ipynb` (Model Training cell)
**Issue:** CLAUDE.md documents that "Round-1 rows have NaN features — drop or impute before fitting." The model notebooks contain no assertion or drop/impute step. The parquet files produced by `notebook_data.ipynb` appear to be pre-cleaned (all three notebooks train successfully), so NaN rows are absent at this time. However, if Phase 1/2 notebooks are re-run with different imputation logic, NaN rows will propagate silently:
- `LogisticRegression.fit()` raises `ValueError: Input X contains NaN` — fails loudly, catchable.
- `RandomForestClassifier.fit()` also raises `ValueError: Input contains NaN`.
- `HistGradientBoostingClassifier.fit()` accepts NaN natively — fails silently by producing a different model than expected.

The silent failure in the GBM case is the dangerous variant: the notebook executes without error while training on a contaminated matrix.

**Fix:** Add an explicit guard in each notebook's Feature Selection cell, immediately after building `X_train` and `X_test`:

```python
nan_count = X_train.isnull().sum().sum()
if nan_count > 0:
    print(f'WARNING: {nan_count} NaN values found — dropping affected rows')
    mask = X_train.notnull().all(axis=1)
    X_train = X_train[mask]
    y_train = y_train[mask]

assert not X_train.isnull().any().any(), 'X_train still contains NaN after drop'
assert not X_test.isnull().any().any(), 'X_test contains NaN — check feature pipeline'
```

---

### WR-02: Naive baseline value (0.4816) contradicts CLAUDE.md documented contract (49.6%)

**File:** `notebook_logistic.ipynb` (Baseline Comparison cell output), `notebook_random_forest.ipynb` (Baseline Comparison cell output)
**Issue:** Both notebooks print `Naive (always HomeWin) accuracy: 0.4816` from their executed outputs. CLAUDE.md states: "Compare against naive 'always Home Win' baseline (49.6% — verified against actual dataset; the ~46% figure is wrong)." The discrepancy is 1.4 percentage points. One of the following is true:

1. The test set (2023–2025, 1,140 rows) has a different home-win rate than the full dataset — this is plausible if recent seasons are less home-dominant.
2. The CLAUDE.md figure was computed on the full dataset (9,165 rows) rather than the held-out test set (1,140 rows).

Either way, the narratives in both notebooks reference "49.6%" in the Results Interpretation markdown cell — but this number does not match what the code actually computes and prints. A reader comparing the prose to the output will find a contradiction.

**Fix:** Update the Results Interpretation markdown cell in both notebooks to cite the actual computed value (printed by the code), or add an explicit note explaining that 49.6% is the full-dataset baseline and 48.16% is the test-set baseline. The mismatch as-is will confuse anyone auditing the results.

---

### WR-03: `VALID_TEAMS` built from `home_team` column only — away-only teams silently excluded from suggestions

**File:** `notebook_gradient_boost.ipynb` (predict_match cell), `notebook_logistic.ipynb` (predict_match cell), `notebook_random_forest.ipynb` (predict_match cell)
**Issue:** `VALID_TEAMS = sorted(combined['home_team'].unique().tolist())` enumerates only teams that appear in the `home_team` column. The validation loop for `away_team` checks `away_last.index` (correctly), but the error message — `Valid teams: {VALID_TEAMS}` — cites `VALID_TEAMS`, which may omit teams that only appear in `away_team`. If such a team name is passed as `home_team`, it won't appear in the error's suggestion list despite being a real team.

In practice, Brazilian Série A teams play both home and away in every season, so `home_team.unique() ≈ away_team.unique()`. However, the dataset spans 2003–2025 and includes relegated/promoted clubs that may have played a partial season predominantly as guests. The assumption is not asserted anywhere.

**Fix:**

```python
VALID_TEAMS = sorted(
    set(combined['home_team'].unique()) | set(combined['away_team'].unique())
)
```

---

## Info

### IN-01: `import numpy as np` is unused in all three notebooks

**File:** `notebook_gradient_boost.ipynb` (Setup cell), `notebook_logistic.ipynb` (Setup cell), `notebook_random_forest.ipynb` (Setup cell)
**Issue:** `numpy` is imported in every notebook's Setup cell but is never referenced in any subsequent cell. All array operations go through pandas or scikit-learn directly.
**Fix:** Remove `import numpy as np` from all three Setup cells. If numpy is needed in the future (e.g., for `np.nan` checks), it can be re-added then.

---

### IN-02: Identical `predict_match` implementation duplicated across all three notebooks with no shared utility

**File:** `notebook_gradient_boost.ipynb`, `notebook_logistic.ipynb`, `notebook_random_forest.ipynb` (predict_match cells)
**Issue:** The `predict_match` function body is identical across all three notebooks except for the bound model variable (`hgb`, `lr`, `rf`). The `home_last`/`away_last` lookup construction, the validation loop, and the vector assembly are copy-pasted verbatim. Bug CR-01 above is a direct consequence of this duplication — fixing it in one notebook requires manually applying the fix in two others, and future regressions are likely.

Since the project constraint is "all deliverables are Jupyter notebooks; no importable `.py` modules," a shared module is not an option. However, within the notebook constraint, documenting that the function is intentionally duplicated (and noting that any change must be propagated to all three) would reduce maintenance risk.

**Fix:** Add a comment block in each notebook's predict_match cell:

```python
# NOTE: This function is duplicated in notebook_logistic.ipynb,
# notebook_random_forest.ipynb, and notebook_gradient_boost.ipynb.
# Any bug fix or enhancement must be applied to all three notebooks.
```

---

_Reviewed: 2026-05-23_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
