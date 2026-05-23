# Phase 4: Advanced Model Notebook + Polish - Context

**Gathered:** 2026-05-23
**Status:** Ready for planning

<domain>
## Phase Boundary

Build `notebook_gradient_boost.ipynb` targeting 65-70% test accuracy, and add submission-quality markdown documentation to all three model notebooks (`notebook_logistic.ipynb`, `notebook_random_forest.ipynb`, `notebook_gradient_boost.ipynb`). The GBM notebook follows the same 8-section structure established in Phase 3. Feature engineering, data pipeline, and the LR/RF model implementations are already complete.

</domain>

<decisions>
## Implementation Decisions

### GBM algorithm variant
- **D-01:** Use `HistGradientBoostingClassifier(class_weight='balanced')` — not `GradientBoostingClassifier`. HGBC supports `class_weight='balanced'` natively (same API pattern as LR and RF notebooks), trains faster, and achieves equivalent or better accuracy. The `sample_weight` workaround required for classic GBC is unnecessary.
- **D-02:** Follow the identical notebook structure as Phase 3 notebooks: Setup → Load Data → Feature Selection → Model Training → Cross-Validation → Evaluation → Baseline Comparison → predict_match.

### Hyperparameter tuning approach
- **D-03:** Manual parameter selection with inline comments explaining each choice — no GridSearchCV or RandomizedSearchCV. Example rationale comments: `learning_rate=0.05  # smaller rate + more trees trades off speed for generalization`, `max_iter=300`, `max_depth=5`. If 65-70% accuracy is not reached, report the actual result honestly in the results cell.
- **D-04:** TSS used for CV score reporting only (same as Phase 3 D-03). Run `cross_val_score` with `TimeSeriesSplit(n_splits=5)` on training set; final evaluation on fixed test parquet.

### Documentation depth (EVAL-04 — applies to all three notebooks)
- **D-05:** Add exactly two new markdown cells per notebook: (1) a **title/intro cell** at the very top explaining the dataset, problem framing, and model approach in 3-4 sentences; (2) a **results interpretation cell** at the bottom explaining what the metrics mean, draw class performance, and how this model compares to the naive 49.6% baseline.
- **D-06:** Section headers (`## Load Data`, `## Model Training`, etc.) are kept as-is — no prose added mid-notebook. The two new cells bookend the notebook.
- **D-07:** The intro cell for each notebook should briefly state the model used and why it was chosen (e.g., for RF: "Random forests handle non-linear interactions well and are robust to overfitting with balanced class weights"; for GBM: "Gradient boosting iteratively corrects residuals and can learn subtle patterns in form data").

### Claude's Discretion
- Exact parameter values for HGBC (learning_rate, max_iter, max_depth, l2_regularization) — choose values that maximize test accuracy while keeping notebook runtime under ~2 minutes
- Exact wording of intro and results interpretation cells
- Whether to include a `random_state` parameter (use 42 for reproducibility, consistent with established pattern)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### GBM notebook inputs (same as Phase 3)
- `dados/feature_matrix_train.parquet` — 8,025 rows × 31 columns. 20 numeric feature columns + 11 base columns. Training set (2003–2022).
- `dados/feature_matrix_test.parquet` — 1,140 rows × 31 columns. Test set (2023–2025).
- `.planning/phases/02-feature-engineering/02-CONTEXT.md` — Exact column names for all 20 features.

### Phase 3 pattern to replicate
- `.planning/phases/03-baseline-model-notebooks/03-CONTEXT.md` — predict_match() implementation (D-05/D-06), evaluation output decisions (D-07/D-08/D-09), notebook structure decisions.
- `notebook_logistic.ipynb` — Reference for exact cell structure, import pattern, predict_match lookup pattern, MPLBACKEND setup.
- `notebook_random_forest.ipynb` — Second reference; confirm both Phase 3 notebooks before adding documentation.

### Requirements
- `.planning/REQUIREMENTS.md` — MODEL-03 (GBM notebook), EVAL-04 (markdown documentation for all three notebooks).
- `.planning/ROADMAP.md` — Phase 4 success criteria: accuracy in 65-70% range, markdown in all three notebooks, predict_match verified.

### Critical gotchas
- `CLAUDE.md` — "Critical Gotchas" section: temporal leakage rules, draw class collapse, baseline is 49.6% (not 46%).

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `notebook_logistic.ipynb` / `notebook_random_forest.ipynb` — Both use identical structure and can be used as direct templates for the GBM notebook. The predict_match() function (cells 16-17 in LR notebook) can be copied verbatim — it is model-agnostic given the fitted model variable.
- `dados/feature_matrix_{train,test}.parquet` — Already exist, NaN-free, ready to load.

### Established Patterns
- `os.environ['MPLBACKEND'] = 'agg'` before matplotlib import — required, already in both Phase 3 notebooks.
- `NON_FEATURE_COLS` list pattern for dropping non-feature columns before fitting.
- `HOME_FEAT_COLS` / `AWAY_FEAT_COLS` split for predict_match lookup.
- Baseline comparison: `naive_acc = (y_test == 'HomeWin').mean()` → 49.6%.
- `labels = ['HomeWin', 'Draw', 'AwayWin']` ordering for classification_report and confusion_matrix.

### Integration Points
- GBM notebook loads from `dados/feature_matrix_{train,test}.parquet` (Phase 2 → Phase 4 contract).
- Documentation additions to LR/RF notebooks are non-breaking — new markdown cells only, no code changes.
- All three notebooks remain independent and self-contained.

</code_context>

<specifics>
## Specific Ideas

- No specific aesthetic requirements beyond the established notebook style.
- HGBC `class_weight='balanced'` keeps the training call consistent with: `model.fit(X_train, y_train)` — no sample_weight plumbing.

</specifics>

<deferred>
## Deferred Ideas

- GridSearchCV / RandomizedSearchCV hyperparameter tuning — not needed; manual params are sufficient for Phase 4.
- Shot/possession features from `campeonato-brasileiro-estatisticas-full.csv` — v2 requirement.
- API-Football live data enrichment — v2.
- Expanding section-level documentation beyond bookend cells — not required for passing EVAL-04.

</deferred>

---

*Phase: 04-advanced-model-notebook-polish*
*Context gathered: 2026-05-23*
