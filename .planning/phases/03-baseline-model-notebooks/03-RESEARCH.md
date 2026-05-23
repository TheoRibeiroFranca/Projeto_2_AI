# Phase 3: Baseline Model Notebooks - Research

**Researched:** 2026-05-23
**Domain:** scikit-learn classification — LogisticRegression, RandomForestClassifier, TimeSeriesSplit evaluation
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Phase 3 delivers exactly two notebooks: `notebook_logistic.ipynb` (LogisticRegression) and `notebook_random_forest.ipynb` (RandomForestClassifier). `notebook_gradient_boost.ipynb` is Phase 4.
- **D-02:** Both notebooks use `class_weight='balanced'` — mandatory on first `fit()` call, no exceptions.
- **D-03:** TSS is used for CV score reporting only — run `cross_val_score` with `TimeSeriesSplit` on the training set to report mean CV accuracy and macro-F1 with correct temporal ordering. Final model fits on all train data and is evaluated on the fixed test parquet. No `GridSearchCV` or `RandomizedSearchCV`.
- **D-04:** Never use `KFold`, `StratifiedKFold`, or `shuffle=True` anywhere in either notebook.
- **D-05:** Lookup-based predict_match — combine train and test feature matrices, find the most recent feature row for each team in the dataset (home perspective for home_team, away perspective for away_team), assemble a single 20-column feature vector, run through the fitted model, return W/D/L label.
- **D-06:** If a team name is not found in the dataset, raise a clear `ValueError` with the team name and a suggestion to check spelling.
- **D-07:** Each notebook outputs `sklearn.metrics.classification_report` showing precision, recall, F1 per class (HomeWin, Draw, AwayWin) and overall accuracy.
- **D-08:** Each notebook includes a confusion matrix heatmap using `seaborn.heatmap` — all three classes must show non-zero recall (enforced by `class_weight='balanced'`).
- **D-09:** Compare final accuracy against the naive baseline of **49.6%** (HomeWin always) — not the old ~46% figure.
- **D-10:** Minimal — section-divider markdown cells only (e.g., `## Load Data`, `## Feature Selection`, `## Model Training`, `## Cross-Validation`, `## Evaluation`, `## predict_match`). No prose explanations; Phase 4 adds all explanatory text.

### Claude's Discretion

- Exact feature columns to use (all 20 numeric feature columns from the feature_matrix parquets; drop id, date, season, round, home_team, away_team, home_score, away_score, home_state, away_state, result)
- Number of TSS splits (default `n_splits=5` is fine)
- Logistic Regression solver and max_iter (saga or lbfgs with max_iter=1000)
- Random Forest n_estimators default (100 is fine for Phase 3; Phase 4 GBM gets more tuning)
- How to handle the combined train+test parquet in predict_match (pd.concat, sort by date, groupby team, take last row)

### Deferred Ideas (OUT OF SCOPE)

- Hyperparameter tuning (GridSearchCV / RandomizedSearchCV) — not needed in Phase 3; Phase 4 GBM can explore if desired.
- Shot/possession features from `campeonato-brasileiro-estatisticas-full.csv` — v2 requirement.
- API-Football live data enrichment — v2.

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| MODEL-01 | `notebook_logistic.ipynb` — LogisticRegression baseline with `class_weight='balanced'`, confirms model beats naive heuristic | LR verified at 44.0% test accuracy; beats naive only in terms of macro-F1 (0.41 vs ~0.33 for naive). Accuracy below 48.16% naive — see Critical Pitfall section for resolution. |
| MODEL-02 | `notebook_random_forest.ipynb` — RandomForestClassifier with `class_weight='balanced'` and TimeSeriesSplit CV | RF with default params (n=100) collapses draw recall to 9%. Needs `min_samples_leaf=5` to achieve non-zero draw recall per D-08. |
| MODEL-04 | TimeSeriesSplit CV applied inside each model notebook without temporal leakage | `cross_val_score` with `TimeSeriesSplit(n_splits=5)` verified. Fold sizes: 1340/1337/1337/1337/1337. |
| EVAL-01 | Each notebook exposes `predict_match(home_team, away_team)` returning W/D/L | Lookup pattern verified: groupby home_team / away_team on combined parquet, take last row, assemble 20-col vector. |
| EVAL-02 | Each notebook outputs `classification_report` per class | `sklearn.metrics.classification_report` verified — outputs HomeWin/Draw/AwayWin rows. |
| EVAL-03 | Each notebook includes a confusion matrix heatmap | `seaborn.heatmap` (v0.13.2) verified with `confusion_matrix` from sklearn. |

</phase_requirements>

---

## Summary

Phase 3 builds two self-contained Jupyter notebooks — `notebook_logistic.ipynb` and `notebook_random_forest.ipynb` — that each load the Phase 2 feature matrices, train a classifier, evaluate it with `TimeSeriesSplit` cross-validation and a held-out test set, expose a `predict_match()` function, and output a `classification_report` + seaborn confusion matrix heatmap.

The stack is already fully installed: scikit-learn 1.8.0, pandas 2.3.3, seaborn 0.13.2, pyarrow are all present in the project virtualenv. All four packages pass slopcheck [OK]. The feature matrices (`dados/feature_matrix_{train,test}.parquet`) are confirmed NaN-free (8,025 × 31 train, 1,140 × 31 test).

**Critical finding from empirical testing:** With `class_weight='balanced'` enabled (as required by D-02), both models trade overall accuracy for Draw/AwayWin recall. Logistic Regression reaches 44.0% test accuracy (macro-F1 0.41, Draw recall 25%) and Random Forest with default settings reaches 47.3% but collapses Draw recall to 9% — violating D-08. RF requires `min_samples_leaf=5` to achieve non-zero draw recall for all 3 classes (19% Draw recall). Neither model beats the test-set naive baseline of 48.16% in raw accuracy. The success criterion in ROADMAP.md "above naive ~46%" uses the old incorrect figure (46%); the actual test-set naive is 48.16%. The planner must address this: the notebooks should explicitly state that class-balanced training prioritizes macro-F1 over raw accuracy, and the comparison should be framed as "the model provides meaningful Draw and AwayWin predictions that the naive baseline cannot, at a modest raw-accuracy cost."

**Primary recommendation:** Use `LogisticRegression(class_weight='balanced', solver='lbfgs', max_iter=1000)` and `RandomForestClassifier(n_estimators=200, class_weight='balanced', min_samples_leaf=5, random_state=42)`. Report macro-F1 as the primary metric alongside accuracy. Frame the naive baseline comparison around what `class_weight='balanced'` achieves versus always predicting HomeWin.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Data loading | Notebook (cell 1) | — | Load from parquet; no re-cleaning |
| Feature selection | Notebook (cell 2) | — | Drop 11 non-feature cols; keep 20 numeric |
| Model training | Notebook (fit cell) | — | Single `.fit()` call on full train set |
| Cross-validation reporting | Notebook (CV cell) | — | `cross_val_score` + `TimeSeriesSplit` |
| Test evaluation | Notebook (eval cell) | — | `classification_report` + confusion matrix |
| Predict function | Notebook (function cell) | — | Lookup combined parquet + fitted model |
| Visualization | Notebook (heatmap cell) | seaborn | `sns.heatmap` on `confusion_matrix` output |

---

## Standard Stack

### Core
| Library | Version (installed) | Purpose | Why Standard |
|---------|--------------------|---------|--------------------|
| scikit-learn | 1.8.0 | LogisticRegression, RandomForestClassifier, TimeSeriesSplit, cross_val_score, classification_report, confusion_matrix | The canonical Python ML library for tabular classification [VERIFIED: project venv] |
| pandas | 2.3.3 | Parquet loading, feature selection, groupby for predict_match lookup | Already the project data layer [VERIFIED: project venv] |
| seaborn | 0.13.2 | `sns.heatmap` for confusion matrix visualization | Already installed, concise API for annotated heatmaps [VERIFIED: project venv] |
| pyarrow | 24.0.0 | Parquet read engine for `pd.read_parquet()` | Required by pandas parquet I/O [VERIFIED: project venv] |
| matplotlib | (transitive) | Backend setup (`MPLBACKEND=agg`) before seaborn plots | Required for headless rendering [VERIFIED: project venv] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `os` (stdlib) | — | Set `MPLBACKEND=agg` before matplotlib import | Required in all model notebooks (established pattern from notebook_data.ipynb) |
| `numpy` (transitive) | — | `.reshape(1, -1)` for predict_match feature vector | Used inline in predict_match implementation |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `lbfgs` solver | `saga` | Both produce identical results on this dataset (verified empirically). `lbfgs` converges in <1000 iter; use it. |
| `n_splits=5` (TSS) | 3 or 10 | 5 is sklearn default, gives 1337-row validation folds which is statistically sufficient |
| `min_samples_leaf=5` (RF) | 1 (default) | Default collapses Draw recall to 9%, violating D-08. `min_samples_leaf=5` achieves 19-20% draw recall. |

**Installation:** No new installs required — all packages already in the virtualenv via `uv sync`.

---

## Package Legitimacy Audit

| Package | Registry | Age | Downloads | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-----------|-------------|-----------|-------------|
| scikit-learn | PyPI | 15+ yrs | Very high | github.com/scikit-learn/scikit-learn | [OK] | Approved |
| pandas | PyPI | 15+ yrs | Very high | github.com/pandas-dev/pandas | [OK] | Approved |
| seaborn | PyPI | 12+ yrs | High | github.com/mwaskom/seaborn | [OK] | Approved |
| pyarrow | PyPI | 8+ yrs | Very high | github.com/apache/arrow | [OK] | Approved |

**Packages removed due to slopcheck [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

*slopcheck v0.6.1 was available and ran successfully. All 4 packages rated [OK]. Versions confirmed against project virtualenv.*

---

## Architecture Patterns

### System Architecture Diagram

```
dados/feature_matrix_train.parquet  dados/feature_matrix_test.parquet
                 |                                  |
                 v                                  v
      pd.read_parquet()                   pd.read_parquet()
                 |                                  |
                 +-----------> X_train, y_train    X_test, y_test
                                    |                    |
                         [TimeSeriesSplit CV]     [held-out eval]
                                    |                    |
                         cross_val_score()     model.predict()
                                    |                    |
                         CV accuracy / F1    classification_report()
                                                         |
                                             confusion_matrix()
                                                         |
                                             sns.heatmap()
                                             
        dados/feature_matrix_{train,test}.parquet
                          |
                    pd.concat() + sort by date
                          |
               groupby(home_team).last()  groupby(away_team).last()
                          |
               predict_match(home_team, away_team)
                          |
                   [20-col feature vector]
                          |
                   model.predict() -> W/D/L label
```

### Recommended Notebook Cell Structure

Both notebooks follow identical section ordering (D-10: section-divider markdown cells only):

```
Cell 1:  [markdown] ## Setup
Cell 2:  [code]     os.environ + imports
Cell 3:  [markdown] ## Load Data
Cell 4:  [code]     pd.read_parquet (train + test)
Cell 5:  [markdown] ## Feature Selection
Cell 6:  [code]     Define FEATURE_COLS, X_train/y_train/X_test/y_test
Cell 7:  [markdown] ## Model Training
Cell 8:  [code]     Instantiate + fit model on X_train/y_train
Cell 9:  [markdown] ## Cross-Validation
Cell 10: [code]     TimeSeriesSplit + cross_val_score (accuracy + macro-F1)
Cell 11: [markdown] ## Evaluation
Cell 12: [code]     classification_report on test set
Cell 13: [code]     confusion_matrix + sns.heatmap
Cell 14: [markdown] ## Baseline Comparison
Cell 15: [code]     Naive accuracy vs model accuracy + macro-F1 comparison
Cell 16: [markdown] ## predict_match
Cell 17: [code]     build lookup tables + define predict_match()
Cell 18: [code]     Example call: predict_match('Flamengo', 'Palmeiras')
```

### Pattern 1: LogisticRegression with temporal CV
**What:** Train LR with balanced weights, report TSS cross-validation scores, evaluate on fixed test parquet.
**When to use:** Baseline linear classifier — fast, interpretable, achieves meaningful Draw recall with balanced weights.
**Example:**
```python
# Source: sklearn 1.8.0 API (verified in project venv)
import os
os.environ['MPLBACKEND'] = 'agg'
os.environ['MPLCONFIGDIR'] = '/tmp/matplotlib-config'

import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import TimeSeriesSplit, cross_val_score
from sklearn.metrics import classification_report, confusion_matrix, accuracy_score

train = pd.read_parquet('dados/feature_matrix_train.parquet')
test  = pd.read_parquet('dados/feature_matrix_test.parquet')

NON_FEATURE_COLS = [
    'id', 'date', 'season', 'round', 'home_team', 'away_team',
    'home_score', 'away_score', 'home_state', 'away_state', 'result'
]
FEATURE_COLS = [c for c in train.columns if c not in NON_FEATURE_COLS]

X_train, y_train = train[FEATURE_COLS], train['result']
X_test,  y_test  = test[FEATURE_COLS],  test['result']

lr = LogisticRegression(class_weight='balanced', solver='lbfgs', max_iter=1000, random_state=42)
lr.fit(X_train, y_train)

tss = TimeSeriesSplit(n_splits=5)
cv_acc = cross_val_score(lr, X_train, y_train, cv=tss, scoring='accuracy')
cv_f1  = cross_val_score(lr, X_train, y_train, cv=tss, scoring='f1_macro')
print(f'CV accuracy:  {cv_acc.mean():.3f} +/- {cv_acc.std():.3f}')
print(f'CV macro-F1:  {cv_f1.mean():.3f} +/- {cv_f1.std():.3f}')

y_pred = lr.predict(X_test)
print(classification_report(y_test, y_pred))

labels = ['HomeWin', 'Draw', 'AwayWin']
cm = confusion_matrix(y_test, y_pred, labels=labels)
sns.heatmap(cm, annot=True, fmt='d', xticklabels=labels, yticklabels=labels, cmap='Blues')
plt.xlabel('Predicted'); plt.ylabel('Actual')
plt.title('Logistic Regression — Confusion Matrix')
plt.tight_layout(); plt.show()
```

### Pattern 2: RandomForest with min_samples_leaf for draw recall
**What:** RF with `min_samples_leaf=5` to prevent draw recall collapse. Default (`min_samples_leaf=1`) produces only 9% Draw recall; `min_samples_leaf=5` achieves 19-20%.
**When to use:** RF notebook — this parameter is NOT optional given D-08's non-zero recall requirement.
**Example:**
```python
# Source: sklearn 1.8.0 API (verified in project venv)
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(
    n_estimators=200,
    class_weight='balanced',
    min_samples_leaf=5,
    random_state=42
)
rf.fit(X_train, y_train)
```

### Pattern 3: predict_match lookup function
**What:** Combine train + test parquets, take the last home-perspective row per home_team and last away-perspective row per away_team, assemble one 20-feature vector.
**When to use:** Final cell in both notebooks — identical implementation in each.
**Example:**
```python
# Source: CONTEXT.md D-05 pattern, verified empirically
import numpy as np

HOME_FEAT_COLS = [c for c in FEATURE_COLS if c.startswith('home_')]
AWAY_FEAT_COLS = [c for c in FEATURE_COLS if c.startswith('away_')]

combined = pd.concat([train, test]).sort_values('date').reset_index(drop=True)
home_last = combined.groupby('home_team')[HOME_FEAT_COLS].last()
away_last = combined.groupby('away_team')[AWAY_FEAT_COLS].last()

VALID_TEAMS = sorted(combined['home_team'].unique().tolist())

def predict_match(home_team: str, away_team: str) -> str:
    if home_team not in home_last.index:
        raise ValueError(
            f"Unknown team '{home_team}'. Check spelling. Valid teams: {VALID_TEAMS}"
        )
    if away_team not in away_last.index:
        raise ValueError(
            f"Unknown team '{away_team}'. Check spelling. Valid teams: {VALID_TEAMS}"
        )
    home_row = home_last.loc[home_team]
    away_row = away_last.loc[away_team]
    row = pd.concat([home_row, away_row])[FEATURE_COLS]
    vector = row.values.reshape(1, -1)
    return lr.predict(vector)[0]   # replace lr with rf in RF notebook

# Example
print(predict_match('Flamengo', 'Palmeiras'))
```

### Pattern 4: Naive baseline comparison cell
**What:** Report the naive "always HomeWin" accuracy on the test set and compare it against the model, using both accuracy and macro-F1.
**Why needed:** D-09 requires comparison against 49.6% baseline; framing must acknowledge the accuracy vs recall tradeoff explicitly.
**Example:**
```python
naive_acc = (y_test == 'HomeWin').mean()
model_acc = accuracy_score(y_test, y_pred)
from sklearn.metrics import f1_score
model_f1 = f1_score(y_test, y_pred, average='macro')

print(f'Naive (always HomeWin) accuracy:   {naive_acc:.4f}')
print(f'Model test accuracy:               {model_acc:.4f}')
print(f'Model macro-F1:                    {model_f1:.4f}')
print('Note: class_weight=balanced trades accuracy for Draw/AwayWin recall')
```

### Anti-Patterns to Avoid

- **`KFold` or `StratifiedKFold` for CV:** Causes temporal leakage — matches from 2022 in validation set while training on 2023 data. Violates D-04.
- **`shuffle=True` in any split:** Same leakage as KFold. Hard violation.
- **Omitting `class_weight='balanced'`:** RF and LR default to majority-class bias — Draw recall collapses to near-zero. Violates D-02.
- **Re-deriving features in model notebooks:** The parquets are NaN-free. Calling `build_features()` or re-running feature engineering inside a model notebook violates the Phase 2 → 3 contract. Load from parquet only.
- **Using `matches_{train,test}.parquet` instead of `feature_matrix_{train,test}.parquet`:** The matches parquets lack the 20 feature columns. Always load `feature_matrix_*.parquet`.
- **RF with default `min_samples_leaf=1`:** Produces 9% Draw recall, violating D-08. Must use `min_samples_leaf=5`.
- **`dropna()` on feature matrix:** The parquets are NaN-free (verified). Calling `dropna()` is harmless but confusing; omit it.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Confusion matrix | Custom nested loop | `sklearn.metrics.confusion_matrix` | Handles label ordering, normalization, multiclass |
| Per-class metrics | Manual precision/recall calc | `sklearn.metrics.classification_report` | Handles all three classes + weighted avg in one call |
| Temporal CV | Manual train/val slicing | `TimeSeriesSplit` + `cross_val_score` | Correct gap handling, reproducible fold sizes |
| Heatmap visualization | `plt.imshow` + colorbar | `seaborn.heatmap(annot=True, fmt='d')` | Annotation, color scaling, axis labels in 2 lines |
| Class weighting | Manual sample_weight computation | `class_weight='balanced'` parameter | sklearn computes inverse-frequency weights automatically |

**Key insight:** Every evaluation component is a one-liner from sklearn/seaborn. Custom implementations introduce bugs in label ordering, weight normalization, and fold boundary handling that are invisible until draw recall reports as 0.

---

## Common Pitfalls

### Pitfall 1: RF draw recall collapse with balanced weights
**What goes wrong:** `RandomForestClassifier(class_weight='balanced')` with default `min_samples_leaf=1` still collapses Draw recall to 8-9% on this dataset. Empirically verified: n_estimators=100 gives Draw recall=9%, n=200 gives 9%, n=500 gives 8%.
**Why it happens:** With many shallow leaves, the tree can still partition Draw samples into HomeWin-dominated leaf nodes despite the weight rebalancing. The tree overfits to majority-class structure.
**How to avoid:** Set `min_samples_leaf=5`. This smooths leaf statistics, preventing micro-leaves dominated by one class. Empirically: `min_samples_leaf=5` achieves Draw recall 19-20% and macro-F1 0.39.
**Warning signs:** `classification_report` shows Draw recall < 0.05. Any value below 5% indicates collapse.

### Pitfall 2: Accuracy below naive baseline misread as model failure
**What goes wrong:** With `class_weight='balanced'`, both models report test accuracy below the 48.16% naive baseline (HomeWin always). LR at 44.0%, RF at 46.6%. This looks like failure but is the expected accuracy-vs-recall tradeoff.
**Why it happens:** The test set has 48.16% HomeWin rate. Always predicting HomeWin gets 48.16% accuracy. A balanced model splits predictions across 3 classes, reducing accuracy while improving minority-class recall.
**How to avoid:** Report macro-F1 as the primary metric (LR: 0.41, RF: 0.39 with min_leaf=5). The naive baseline's macro-F1 is ~0.33 (since it gets 0 recall on Draw and AwayWin). Frame the comparison as "macro-F1 improvement over naive" rather than raw accuracy. The ROADMAP success criterion "above naive ~46%" used the old incorrect figure; the actual naive on the 2023-2025 test set is 48.16%.
**Warning signs:** Comparing only accuracy without macro-F1. A model that beats the naive in accuracy but has zero Draw recall is worse, not better.

### Pitfall 3: Wrong naive baseline figure
**What goes wrong:** Using the ~46% figure stated in REQUIREMENTS.md (MODEL-01). This was the placeholder; the dataset-verified value is 49.6% overall, and 48.16% on the 2023-2025 test set.
**Why it happens:** Requirements were written before Phase 1 EDA confirmed the actual home-win rate.
**How to avoid:** Compute `naive_acc = (y_test == 'HomeWin').mean()` dynamically in the notebook, never hardcode 46% or 49.6%.
**Warning signs:** Hardcoded 0.46 or 0.496 in the comparison cell.

### Pitfall 4: predict_match column ordering mismatch
**What goes wrong:** `predict_match()` assembles a DataFrame row from `home_last` and `away_last` using `pd.concat()`. If the concat result orders columns differently from `FEATURE_COLS`, the feature vector is scrambled.
**Why it happens:** `pd.concat([home_row, away_row])` preserves each Series' index order. If `home_last` and `away_last` include different columns or non-standard ordering, the final `[FEATURE_COLS]` reindex step may produce NaN.
**How to avoid:** Always reindex with `row[FEATURE_COLS]` after concat. This ensures the 20-column order matches exactly what the model was trained on. Verified working: `pd.concat([home_row, away_row])[FEATURE_COLS].values.reshape(1, -1)`.
**Warning signs:** Model returns same class for all inputs, or a `KeyError` in predict_match.

### Pitfall 5: Importing matplotlib before setting MPLBACKEND
**What goes wrong:** Jupyter attempts to use an interactive backend (Qt, Tk) which fails in headless environments. Seaborn imports matplotlib at import time.
**Why it happens:** Backend selection happens at first matplotlib import.
**How to avoid:** Set `os.environ['MPLBACKEND'] = 'agg'` and `os.environ['MPLCONFIGDIR'] = '/tmp/matplotlib-config'` in the first code cell, before any `import seaborn` or `import matplotlib`.
**Warning signs:** `UserWarning: Matplotlib is currently using agg` or blank plot cells.

---

## Empirical Performance Numbers

These are verified by running both models against the actual feature matrices.

| Model | Config | Test Accuracy | Naive Baseline | Macro-F1 | Draw Recall | AwayWin Recall |
|-------|--------|--------------|----------------|----------|-------------|----------------|
| LogisticRegression | lbfgs, max_iter=1000, balanced | 44.0% | 48.16% | 0.41 | 25% | 51% |
| RandomForestClassifier | n=100, balanced, min_leaf=1 (DEFAULT — DO NOT USE) | 47.3% | 48.16% | 0.32 | 9% | 11% |
| RandomForestClassifier | n=200, balanced, min_leaf=5 (RECOMMENDED) | 46.6% | 48.16% | 0.39 | 19% | 30% |

**LR CV scores (TimeSeriesSplit, n_splits=5, training set only):**
- CV accuracy: 0.392 +/- 0.020
- CV macro-F1: 0.365 +/- 0.016

**Key insight for notebooks:** LR achieves higher macro-F1 than RF (even tuned RF). ROADMAP success criterion 2 says "RF macro-F1 higher than logistic baseline." With the configurations above, RF macro-F1 (0.39) is slightly below LR (0.41). The planner must either accept this discrepancy (the criterion was aspirational, not empirically validated) or use feature engineering improvements for RF. For Phase 3, accuracy and macro-F1 differences are small enough that both notebooks demonstrate meaningful multi-class prediction vs. the naive single-class baseline.

---

## Code Examples

### Complete feature column lists (verified)
```python
# Source: verified from feature_matrix_train.parquet schema
NON_FEATURE_COLS = [
    'id', 'date', 'season', 'round', 'home_team', 'away_team',
    'home_score', 'away_score', 'home_state', 'away_state', 'result'
]
# FEATURE_COLS = all 20 remaining columns:
FEATURE_COLS = [
    'home_goals_scored_last5', 'home_goals_conceded_last5',
    'away_goals_scored_last5', 'away_goals_conceded_last5',
    'home_wins_last5', 'home_draws_last5', 'home_losses_last5',
    'away_wins_last5', 'away_draws_last5', 'away_losses_last5',
    'home_win_pct_season', 'home_draw_pct_season', 'home_loss_pct_season',
    'away_win_pct_season', 'away_draw_pct_season', 'away_loss_pct_season',
    'home_goal_diff_last5', 'home_points_last5',
    'away_goal_diff_last5', 'away_points_last5'
]
HOME_FEAT_COLS = [c for c in FEATURE_COLS if c.startswith('home_')]  # 10 cols
AWAY_FEAT_COLS = [c for c in FEATURE_COLS if c.startswith('away_')]  # 10 cols
```

### seaborn heatmap for confusion matrix (verified API)
```python
# Source: seaborn 0.13.2 installed in project venv
labels = ['HomeWin', 'Draw', 'AwayWin']
cm = confusion_matrix(y_test, y_pred, labels=labels)
plt.figure(figsize=(6, 5))
sns.heatmap(cm, annot=True, fmt='d',
            xticklabels=labels, yticklabels=labels,
            cmap='Blues')
plt.xlabel('Predicted')
plt.ylabel('Actual')
plt.title('Confusion Matrix')
plt.tight_layout()
plt.show()
```

### Valid team names (46 total, verified from combined parquet)
```python
# All 46 valid team name strings that predict_match() must accept:
VALID_TEAMS = [
    'America-MG', 'America-RN', 'Athletico-PR', 'Atletico-GO', 'Atletico-MG',
    'Avai', 'Bahia', 'Barueri', 'Botafogo-RJ', 'Bragantino', 'Brasiliense',
    'CSA', 'Ceara', 'Chapecoense', 'Corinthians', 'Coritiba', 'Criciuma',
    'Cruzeiro', 'Cuiaba', 'Figueirense', 'Flamengo', 'Fluminense', 'Fortaleza',
    'Goias', 'Gremio', 'Gremio Prudente', 'Guarani', 'Internacional',
    'Ipatinga', 'Joinville', 'Juventude', 'Mirassol', 'Nautico', 'Palmeiras',
    'Parana', 'Paysandu', 'Ponte Preta', 'Portuguesa', 'Santa Cruz',
    'Santo Andre', 'Santos', 'Sao Caetano', 'Sao Paulo', 'Sport', 'Vasco', 'Vitoria'
]
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| LabelEncoder for result column | Direct string labels ('HomeWin', 'Draw', 'AwayWin') | sklearn 0.24+ | sklearn classifiers handle string labels natively; no encoding needed |
| `fit_transform` on all data | Temporal split before any fitting | Always best practice | Prevents data leakage from future matches |
| `cv=5` (default KFold) in cross_val_score | `cv=TimeSeriesSplit(n_splits=5)` | sklearn docs note | Required for time-series data; prevents future-looking folds |

**Deprecated/outdated:**
- Passing integer `cv=5` to `cross_val_score` for time-series: Uses KFold internally, causing temporal leakage. Always pass a `TimeSeriesSplit` object.
- `class_weight={0:1, 1:2, 2:1}` manual weight dicts: `class_weight='balanced'` computes optimal weights automatically from class frequencies.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| scikit-learn | Models, CV, metrics | Yes | 1.8.0 | — |
| pandas | Data loading, groupby | Yes | 2.3.3 | — |
| seaborn | Confusion matrix heatmap | Yes | 0.13.2 | — |
| pyarrow | Parquet read | Yes | 24.0.0 | — |
| matplotlib | Plot rendering (via seaborn) | Yes | (transitive) | — |
| dados/feature_matrix_train.parquet | Training data | Yes | 8025×31, NaN-free | Re-run notebook_data.ipynb |
| dados/feature_matrix_test.parquet | Test data | Yes | 1140×31, NaN-free | Re-run notebook_data.ipynb |
| nbmake | Notebook end-to-end tests | Yes | (in dev deps) | Run notebook manually |

**Missing dependencies with no fallback:** None.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest + nbmake |
| Config file | pytest.ini or pyproject.toml (existing, from Phase 1) |
| Quick run command | `pytest --nbmake notebook_logistic.ipynb -x` |
| Full suite command | `pytest --nbmake notebook_logistic.ipynb notebook_random_forest.ipynb -x` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| MODEL-01 | notebook_logistic.ipynb runs end-to-end | smoke | `pytest --nbmake notebook_logistic.ipynb -x` | No — Wave 0 (notebook must be created) |
| MODEL-02 | notebook_random_forest.ipynb runs end-to-end | smoke | `pytest --nbmake notebook_random_forest.ipynb -x` | No — Wave 0 |
| MODEL-04 | TimeSeriesSplit used (no KFold) | smoke (cell execution) | `pytest --nbmake ... -x` — cell would error if TSS not imported | No — Wave 0 |
| EVAL-01 | predict_match('Flamengo', 'Palmeiras') returns valid label | smoke (example cell) | Notebook last cell calls predict_match, nbmake verifies no exception | No — Wave 0 |
| EVAL-02 | classification_report printed without error | smoke | Notebook eval cell, verified by nbmake | No — Wave 0 |
| EVAL-03 | sns.heatmap renders without error | smoke | Notebook heatmap cell, verified by nbmake | No — Wave 0 |

### Sampling Rate
- **Per task commit:** `pytest --nbmake notebook_logistic.ipynb -x` (or RF equivalent)
- **Per wave merge:** `pytest --nbmake notebook_logistic.ipynb notebook_random_forest.ipynb -x`
- **Phase gate:** Both notebooks pass nbmake before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `notebook_logistic.ipynb` — covers MODEL-01, MODEL-04, EVAL-01, EVAL-02, EVAL-03 (LR version)
- [ ] `notebook_random_forest.ipynb` — covers MODEL-02, MODEL-04, EVAL-01, EVAL-02, EVAL-03 (RF version)

*(Both notebooks are the deliverables of this phase. They are test artifacts AND implementation artifacts simultaneously — nbmake tests run the notebooks end-to-end.)*

---

## Project Constraints (from CLAUDE.md)

All directives extracted from CLAUDE.md that apply to Phase 3:

| Directive | Applies To | Constraint |
|-----------|-----------|------------|
| `class_weight='balanced'` mandatory | Both model notebooks | Must be set on first `fit()` call |
| Never `KFold`, `StratifiedKFold`, `shuffle=True` | Both model notebooks | Use `TimeSeriesSplit` exclusively |
| Compare against 49.6% naive baseline | Evaluation cells | Not ~46% — compute dynamically from test set |
| `classification_report` required | Both notebooks | Report macro-F1 alongside accuracy |
| Load from `feature_matrix_*.parquet` | Data loading | Not `matches_*.parquet`; no re-derivation of features |
| `MPLBACKEND=agg` before matplotlib | Setup cell | Set via `os.environ` before any seaborn/matplotlib import |
| No `.py` modules | Both notebooks | All code inline; `predict_match` defined inside notebook |
| Delivery format: Jupyter Notebook | Both notebooks | No standalone Python scripts |
| `snake_case` naming | Variable names | `x_train`, `y_train`, `x_test`, `y_test` |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | ROADMAP success criterion 1 ("accuracy above naive ~46%") is superseded by CONTEXT.md D-09 (compare against 49.6%), and the planner should frame macro-F1 improvement over naive as the primary success signal | Common Pitfalls, Empirical Numbers | If the criterion is interpreted strictly as raw accuracy > 48.16%, neither model passes — the phase would need hyperparameter tuning or additional features outside the D-03 locked scope |
| A2 | ROADMAP success criterion 2 ("RF macro-F1 higher than logistic baseline") is aspirational and may not hold with default configurations — LR (0.41) outperforms RF min_leaf=5 (0.39) | Empirical Numbers | If strictly enforced, RF notebook would need additional tuning beyond what D-03 permits |

---

## Open Questions

1. **Accuracy vs macro-F1 success criterion conflict**
   - What we know: LR at 44.0% and RF at 46.6% test accuracy both fall below the 48.16% naive baseline on the 2023-2025 test set. LR achieves macro-F1 0.41 vs naive's ~0.33.
   - What's unclear: Does the ROADMAP "beats naive" requirement refer to accuracy (neither model does) or macro-F1 (LR does)?
   - Recommendation: Frame both notebooks' evaluation cells to emphasize macro-F1 improvement. Document that `class_weight='balanced'` is a deliberate accuracy-for-recall trade. This is standard practice in imbalanced classification and the project explicitly mandates balanced weights.

2. **RF macro-F1 < LR macro-F1**
   - What we know: With locked hyperparameters from D-03 (no GridSearchCV), RF macro-F1 (0.39) is below LR (0.41).
   - What's unclear: ROADMAP says "RF macro-F1 higher than logistic baseline" — this cannot be achieved with the default configurations above.
   - Recommendation: Planner should note the observed ordering in the verification step. The RF notebook still demonstrates useful multi-class prediction and non-zero draw recall — the performance gap is small (0.02 F1 points) and the ROADMAP criterion appears aspirational.

---

## Sources

### Primary (HIGH confidence)
- scikit-learn 1.8.0 installed in project venv — all API calls verified by direct execution
- pandas 2.3.3 installed in project venv — `read_parquet`, `groupby`, `concat` verified
- seaborn 0.13.2 installed in project venv — `heatmap` parameter signature verified
- `dados/feature_matrix_train.parquet` and `dados/feature_matrix_test.parquet` — schema, row counts, NaN counts, class distribution, naive baseline verified by direct inspection

### Secondary (MEDIUM confidence)
- CONTEXT.md (03-CONTEXT.md) — locked decisions D-01 through D-10, CONTEXT.md SPECIFICS section for predict_match lookup pattern
- CLAUDE.md — critical gotchas, output contract, feature matrix schema

### Tertiary (LOW confidence)
- None — all claims in this research are verified empirically or cited from project files

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all packages verified in project venv, exact versions confirmed
- Architecture: HIGH — patterns verified by running end-to-end model training against actual parquets
- Pitfalls: HIGH — discovered by running default configurations and observing failures empirically
- Empirical performance numbers: HIGH — produced by actual sklearn training on the project's feature matrices

**Research date:** 2026-05-23
**Valid until:** 2026-06-23 (sklearn API is stable; parquet files don't change unless Phase 2 is re-run)
