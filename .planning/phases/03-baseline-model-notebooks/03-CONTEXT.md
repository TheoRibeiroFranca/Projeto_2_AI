# Phase 3: Baseline Model Notebooks - Context

**Gathered:** 2026-05-23
**Status:** Ready for planning

<domain>
## Phase Boundary

Build and evaluate two separate model notebooks — Logistic Regression and Random Forest — that each beat the naive home-win baseline (49.6%), expose a `predict_match(home_team, away_team)` function, and output per-class evaluation metrics. Gradient Boosting and documentation polish are separate phases.

</domain>

<decisions>
## Implementation Decisions

### Model scope
- **D-01:** Phase 3 delivers exactly two notebooks: `notebook_logistic.ipynb` (LogisticRegression) and `notebook_random_forest.ipynb` (RandomForestClassifier). `notebook_gradient_boost.ipynb` is Phase 4.
- **D-02:** Both notebooks use `class_weight='balanced'` — mandatory on first `fit()` call, no exceptions.

### TimeSeriesSplit usage
- **D-03:** TSS is used for **CV score reporting only** — run `cross_val_score` with `TimeSeriesSplit` on the training set to report mean CV accuracy and macro-F1 with correct temporal ordering. Final model fits on all train data and is evaluated on the fixed test parquet. No `GridSearchCV` or `RandomizedSearchCV`.
- **D-04:** Never use `KFold`, `StratifiedKFold`, or `shuffle=True` anywhere in either notebook.

### `predict_match()` implementation
- **D-05:** Lookup-based approach — combine train and test feature matrices, find the most recent feature row for each team in the dataset (home perspective for home_team, away perspective for away_team), assemble a single 20-column feature vector, run through the fitted model, return W/D/L label.
- **D-06:** If a team name is not found in the dataset, raise a clear `ValueError` with the team name and a suggestion to check spelling.

### Evaluation output
- **D-07:** Each notebook outputs `sklearn.metrics.classification_report` showing precision, recall, F1 per class (HomeWin, Draw, AwayWin) and overall accuracy.
- **D-08:** Each notebook includes a confusion matrix heatmap using `seaborn.heatmap` — all three classes must show non-zero recall (enforced by `class_weight='balanced'`).
- **D-09:** Compare final accuracy against the naive baseline of **49.6%** (HomeWin always) — not the old ~46% figure.

### Notebook documentation in Phase 3
- **D-10:** Minimal — section-divider markdown cells only (e.g., `## Load Data`, `## Feature Selection`, `## Model Training`, `## Cross-Validation`, `## Evaluation`, `## predict_match`). No prose explanations; Phase 4 adds all explanatory text.

### Claude's Discretion
- Exact feature columns to use (all 20 numeric feature columns from the feature_matrix parquets; drop id, date, season, round, home_team, away_team, home_score, away_score, home_state, away_state, result)
- Number of TSS splits (default `n_splits=5` is fine)
- Logistic Regression solver and max_iter (saga or lbfgs with max_iter=1000)
- Random Forest n_estimators default (100 is fine for Phase 3; Phase 4 GBM gets more tuning)
- How to handle the combined train+test parquet in predict_match (pd.concat, sort by date, groupby team, take last row)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Feature contract (inputs to model notebooks)
- `dados/feature_matrix_train.parquet` — 8,025 rows × 31 columns, NaN-free. 20 numeric feature columns + 11 base columns. Training set (2003–2022).
- `dados/feature_matrix_test.parquet` — 1,140 rows × 31 columns, NaN-free. Test set (2023–2025).
- `.planning/phases/02-feature-engineering/02-CONTEXT.md` — Exact column names for all 20 features, imputation decisions (NaN already handled — parquets are NaN-free).

### Requirements
- `.planning/REQUIREMENTS.md` — MODEL-01, MODEL-02, MODEL-04, EVAL-01, EVAL-02, EVAL-03 (Phase 3 requirements with exact acceptance criteria).
- `.planning/ROADMAP.md` — Phase 3 success criteria (5 items): accuracy above 49.6%, RF macro-F1 > logistic, TSS only, predict_match present, classification_report + confusion matrix with non-zero draw recall.

### Critical gotchas
- `CLAUDE.md` — "Critical Gotchas" section: temporal leakage rules, draw class collapse, team name normalization, evaluation requirements.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `notebook_data.ipynb` — Phase 1+2 notebook. Shows notebook style (section markdown cells, MPLBACKEND=agg setup, Portuguese-flavor variable names for legacy sections). Model notebooks follow the same style but use English throughout.
- `dados/feature_matrix_train.parquet` / `dados/feature_matrix_test.parquet` — 31-column parquets, NaN-free, ready to load with `pd.read_parquet()`.

### Established Patterns
- `MPLBACKEND=agg` set via `os.environ` before matplotlib import — required for headless rendering.
- Temporal split: train ≤ 2022, test ≥ 2023. Never re-split inside model notebooks.
- `result` column encoding: `'HomeWin'` / `'Draw'` / `'AwayWin'` — must match exactly; LabelEncoder or direct string comparison both work.
- No `.py` modules — all code is inline in the notebook. Both model notebooks are self-contained (some code duplication between them is acceptable).

### Integration Points
- Both model notebooks load from `dados/feature_matrix_{train,test}.parquet` — this is the Phase 2 → Phase 3 contract.
- `predict_match()` in each notebook uses the fitted model + the combined feature matrix for team lookups; it does not call `build_features()` from `notebook_data.ipynb`.
- The `result` column is the target variable; all other non-feature columns (id, date, season, round, home_team, away_team, home_score, away_score, home_state, away_state) are dropped before fitting.

</code_context>

<specifics>
## Specific Ideas

- No specific UI or aesthetic requirements beyond what's in REQUIREMENTS.md.
- `predict_match()` lookup: `pd.concat([train, test]).sort_values('date').groupby('home_team').last()` for home stats, same for away. Assemble one row from home_team's last home stats + away_team's last away stats.

</specifics>

<deferred>
## Deferred Ideas

- Hyperparameter tuning (GridSearchCV / RandomizedSearchCV) — not needed in Phase 3; Phase 4 GBM can explore if desired.
- Shot/possession features from `campeonato-brasileiro-estatisticas-full.csv` — v2 requirement.
- API-Football live data enrichment — v2.

</deferred>

---

*Phase: 03-baseline-model-notebooks*
*Context gathered: 2026-05-23*
