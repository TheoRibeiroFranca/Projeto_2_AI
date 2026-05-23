# Phase 3: Baseline Model Notebooks - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions captured in 03-CONTEXT.md — this log preserves the discussion.

**Date:** 2026-05-23
**Phase:** 03-baseline-model-notebooks
**Mode:** discuss (default)
**Areas discussed:** predict_match() implementation, TimeSeriesSplit role, Phase 3 documentation depth

---

## Areas Discussed

### predict_match() implementation

| Question | Options Presented | User Selection |
|----------|------------------|----------------|
| How should predict_match() get feature values? | Lookup most recent row (Recommended) / Compute live from match history | Lookup most recent row |

**Notes:** Lookup approach uses combined train+test feature matrix, finds most recent row per team (home perspective for home_team, away perspective for away_team). If team not found → raise ValueError. ~10 lines of code vs ~50 for compute-live.

---

### TimeSeriesSplit role

| Question | Options Presented | User Selection |
|----------|------------------|----------------|
| What should TSS do in each notebook? | CV score reporting only (Recommended) / Hyperparameter tuning via RandomizedSearchCV | CV score reporting only |

**Notes:** `cross_val_score` with TSS on training set to show temporal-correct CV methodology. Final model fits on all train data, evaluated on fixed test parquet. No hyperparameter search in Phase 3.

---

### Phase 3 documentation depth

| Question | Options Presented | User Selection |
|----------|------------------|----------------|
| How much markdown documentation in Phase 3? | Minimal — section headers only (Recommended) / Partial — brief explanations per section | Minimal — section headers only |

**Notes:** Phase 4 owns all explanatory prose. Phase 3 notebooks get section-divider markdown cells only (`## Load Data`, `## Model Training`, etc.). Clean separation of concerns between phases.

---

## Decisions at Claude's Discretion

- Feature columns: all 20 numeric features; drop base columns before fitting
- TSS n_splits: default 5
- Logistic solver: saga or lbfgs, max_iter=1000
- RF n_estimators: 100 (Phase 3 baseline; no tuning)
- predict_match lookup: `pd.concat([train, test]).sort_values('date').groupby(team_col).last()`

## Deferred Ideas

- None emerged during discussion — scope stayed within Phase 3 boundary.

---

*Discussion completed: 2026-05-23*
