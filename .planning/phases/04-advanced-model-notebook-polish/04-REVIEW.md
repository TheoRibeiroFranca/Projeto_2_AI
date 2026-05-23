---
phase: 04-advanced-model-notebook-polish
reviewed: 2026-05-23T00:00:00Z
depth: standard
files_reviewed: 5
files_reviewed_list:
  - CLAUDE.md
  - notebook_data.ipynb
  - notebook_gradient_boost.ipynb
  - notebook_logistic.ipynb
  - notebook_random_forest.ipynb
findings:
  critical: 3
  warning: 3
  info: 2
  total: 8
status: issues_found
---

# Phase 4: Code Review Report

**Reviewed:** 2026-05-23
**Depth:** standard
**Files Reviewed:** 5
**Status:** issues_found

## Summary

Five files were reviewed: the data pipeline notebook, three model notebooks (Logistic Regression, Random Forest, Gradient Boosting), and CLAUDE.md.

The Logistic Regression and Random Forest notebooks are structurally compliant: correct temporal split, `class_weight='balanced'`, `TimeSeriesSplit`, `classification_report`, and baseline comparison are all present and correct.

The Gradient Boosting notebook is critically broken: `HistGradientBoostingClassifier` is instantiated **without** `class_weight='balanced'`, producing complete Draw and AwayWin class collapse — 0% recall for both minority classes, model accuracy identical to the naive baseline. Worse, the Results Interpretation markdown cell falsely claims the model was trained with `class_weight='balanced'`, directly contradicting the visible classification_report output.

The data pipeline notebook has a silent normalization failure: `canonical_names = {}` is never populated, so the mandated team-name variant mapping is a no-op. All three model notebooks share a `predict_match` venue-split staleness bug and a `VALID_TEAMS` incompleteness issue.

---

## Critical Issues

### CR-01: `class_weight='balanced'` Missing from GBM — Complete Draw/AwayWin Collapse

**File:** `notebook_gradient_boost.ipynb` (Model Training cell)

**Issue:** `HistGradientBoostingClassifier` is constructed without `class_weight='balanced'`. The executed output confirms total class collapse: Draw recall is 0% (0/298 predicted) and AwayWin recall is 0% (0/293 predicted). The model predicts HomeWin for every test example, making it exactly equivalent to the naive baseline (model accuracy 0.4816 = naive accuracy 0.4816). CLAUDE.md's "Critical Gotchas" section explicitly mandates `class_weight='balanced'` before the first `fit()` call and identifies draw class collapse as the top risk. The CV macro-F1 of 0.221 (below 0.33 random-chance floor) confirms the collapse extends through cross-validation as well.

**Current code:**
```python
hgb = HistGradientBoostingClassifier(
    learning_rate=0.001,
    max_iter=300,
    max_depth=5,
    random_state=42,
)
```

**Fix:**
```python
hgb = HistGradientBoostingClassifier(
    learning_rate=0.001,
    max_iter=300,
    max_depth=5,
    class_weight='balanced',
    random_state=42,
)
```

---

### CR-02: Results Interpretation Cell Contains Factually False Claim About `class_weight`

**File:** `notebook_gradient_boost.ipynb` (Results Interpretation markdown cell — last cell, id `34262e0e-0be3-47b4-bc40-b72763c518e0`)

**Issue:** The markdown states: "with `class_weight='balanced'`, raw accuracy (~42%) is expected to fall below this baseline because the model redistributes predictions toward Draw and AwayWin to avoid class collapse." This is false in two ways. First, the model was NOT trained with `class_weight='balanced'` (see CR-01). Second, the actual output shows 48.2% accuracy (not ~42%) and 0% recall for Draw and AwayWin — the opposite of what the text describes. Any reviewer who reads this cell and compares it to the classification_report immediately above will find a direct factual contradiction. This is not merely stale documentation; the cell is actively misleading about whether the project's primary correctness requirement was met.

**Fix:** After resolving CR-01 and re-running the notebook, rewrite this cell to reflect the actual outputs. The Results Interpretation text in `notebook_logistic.ipynb` is a safe template (it makes no false causal claims about `class_weight`).

---

### CR-03: `canonical_names` Dictionary Is Always Empty — Team Normalization Silently Skipped

**File:** `notebook_data.ipynb` (Section 4 cell — derive_result)

**Issue:** `canonical_names = {}` is defined as an empty dict and is never populated. The subsequent `df[col].replace(canonical_names)` calls in both Section 4 and Section 6 are no-ops. CLAUDE.md explicitly mandates: "Build a canonical name dict before any `groupby` — 20+ seasons produce many variants (e.g. 'Atletico-MG' vs 'Atletico MG')." The top-10 club assertion in Section 6 passes only because it checks the already-variant spellings (e.g., `'Atletico-MG'`) — it cannot detect inconsistencies between variants. Downstream model notebooks inherit these inconsistent names; a team appearing under two spellings will produce two separate rows in `home_last`/`away_last` inside `predict_match`, silently returning wrong feature values or raising a `KeyError` when the canonical spelling is used.

**Fix:** Populate `canonical_names` with known multi-season variant mappings before the first `.replace()` call. Discover actual variants by inspecting `df['mandante'].value_counts()` before normalization:
```python
canonical_names = {
    'Atletico MG': 'Atletico-MG',
    'Atletico Mineiro': 'Atletico-MG',
    'Atletico Goianiense': 'Atletico-GO',
    'Athletico Paranaense': 'Athletico-PR',
    # ... enumerate all multi-season variants found in value_counts()
}
for col in ['mandante', 'visitante', 'vencedor']:
    df[col] = df[col].replace(canonical_names)
```

---

## Warnings

### WR-01: `predict_match` Uses Venue-Split Lookups — Returns Stale Features for Teams Whose Last Match Was Away

**File:** `notebook_logistic.ipynb`, `notebook_random_forest.ipynb`, `notebook_gradient_boost.ipynb` (predict_match cell in each)

**Issue:** All three notebooks build:
```python
home_last = combined.groupby('home_team')[HOME_FEAT_COLS].last()
away_last = combined.groupby('away_team')[AWAY_FEAT_COLS].last()
```
`HOME_FEAT_COLS` contains only columns prefixed `home_`. `.groupby('home_team').last()` returns the last row where the team appeared as the *home side*. For a team whose most recent match was an away fixture, `home_last.loc[team]` returns their form values from an earlier home match — potentially many weeks old. The `predict_match` call then passes stale rolling-window values into the model, silently producing predictions on out-of-date features. The same staleness applies symmetrically to `away_last`.

**Fix:** Build a venue-agnostic "last match per team" lookup using the long-format view:
```python
# Derive last row per team from either venue
home_view = combined[['date', 'home_team'] + HOME_FEAT_COLS].rename(columns={'home_team': 'team'})
away_view = combined[['date', 'away_team'] + AWAY_FEAT_COLS].rename(columns={'away_team': 'team'})
# Rename away_* columns to home_* for a unified schema
away_view.columns = ['date', 'team'] + HOME_FEAT_COLS

team_last = (
    pd.concat([home_view, away_view])
    .sort_values('date')
    .groupby('team')
    .last()
)
# Then in predict_match:
# home_row = team_last.loc[home_team]  (home features from their most recent match)
# away_row = team_last.loc[away_team]  (away features from their most recent match)
```

---

### WR-02: GBM `learning_rate=0.001` Is Excessively Low — Model Will Likely Underfit Even After CR-01 Fix

**File:** `notebook_gradient_boost.ipynb` (Model Training cell)

**Issue:** `learning_rate=0.001` combined with `max_iter=300` is a highly conservative configuration. scikit-learn's default is `learning_rate=0.1`; at 0.001 the model requires approximately 100× more iterations to achieve equivalent effective learning. On a dataset of 8,025 rows with 20 features, 300 iterations at this rate is severely insufficient for convergence. The CV macro-F1 of 0.221 is well below Logistic Regression's 0.367 — on the same features, a properly configured GBM should outperform LR. After fixing CR-01, this hyperparameter combination will likely still yield a markedly underperforming model relative to what HistGBT is capable of on this data.

**Fix:** Use a learning rate closer to the scikit-learn default and add early stopping:
```python
hgb = HistGradientBoostingClassifier(
    learning_rate=0.05,
    max_iter=500,
    max_depth=4,
    class_weight='balanced',
    random_state=42,
    early_stopping=True,
    validation_fraction=0.1,
    n_iter_no_change=20,
)
```

---

### WR-03: `VALID_TEAMS` Built from `home_team` Column Only — Away-Only Teams Excluded from Error Suggestions

**File:** `notebook_logistic.ipynb`, `notebook_random_forest.ipynb`, `notebook_gradient_boost.ipynb` (predict_match cell)

**Issue:** `VALID_TEAMS = sorted(combined['home_team'].unique().tolist())` enumerates only teams that appear in the `home_team` column. The validation loop for `away_team` correctly checks `away_last.index`, but the error message cites `VALID_TEAMS` — which may omit clubs that appear only in the `away_team` column. Across 20+ seasons with promotion/relegation, partial-season clubs may skew heavily away or home. A user who passes one of these teams as `home_team` will receive a `ValueError` whose `Valid teams:` list silently omits the very team they tried to use (spelled differently), making the suggestion set incomplete.

**Fix:**
```python
VALID_TEAMS = sorted(
    set(combined['home_team'].unique()) | set(combined['away_team'].unique())
)
```

---

## Info

### IN-01: CLAUDE.md Gradient Boost Entry Still Commented Out as "Not Yet Created"

**File:** `CLAUDE.md` (line 33)

**Issue:** The Setup section still contains:
```bash
# jupyter notebook notebook_gradient_boost.ipynb  # Gradient Boosting — Phase 4, ainda não criado
```
The notebook now exists. The comment says "ainda não criado" (not yet created), which is incorrect. Phase status on line 9 also reads "Phase 4 (GBM + polish) pending".

**Fix:** Uncomment the launch command, remove the parenthetical, and mark Phase 4 complete:
```bash
jupyter notebook notebook_gradient_boost.ipynb   # Gradient Boosting
```
Update line 9: `Phase 4 (GBM + polish) ✓`

---

### IN-02: `UndefinedMetricWarning` in GBM Evaluation Output Not Suppressed

**File:** `notebook_gradient_boost.ipynb` (Evaluation cell — `classification_report` call)

**Issue:** The executed cell output includes three `UndefinedMetricWarning` lines because precision is ill-defined for Draw and AwayWin when the model predicts zero samples for those classes. These warnings are a direct symptom of CR-01 but will recur after any partial fix if recall remains low for any class. The `classification_report` function accepts a `zero_division` parameter to suppress this warning cleanly.

**Fix:**
```python
print(classification_report(
    y_test, y_pred,
    labels=['HomeWin', 'Draw', 'AwayWin'],
    target_names=['HomeWin', 'Draw', 'AwayWin'],
    zero_division=0,
))
```

---

_Reviewed: 2026-05-23_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
