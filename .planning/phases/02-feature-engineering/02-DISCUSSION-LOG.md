# Phase 2: Feature Engineering - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions captured in 02-CONTEXT.md — this log preserves the discussion.

**Date:** 2026-05-22
**Phase:** 02-feature-engineering
**Mode:** discuss (default)
**Areas discussed:** Notebook location, Rolling window scope, NaN imputation strategy

---

## Discussion

### Notebook location

**Question:** Where should the Phase 2 feature engineering code live?

| Option | Description |
|--------|-------------|
| Extend notebook_data.ipynb | Add Phase 2 cells after Phase 1's parquet-save block. One pipeline notebook. |
| New notebook_features.ipynb | Dedicated feature notebook reads Phase 1 parquet, saves feature parquet. |
| Inline in each model notebook | Each model notebook computes its own features. build_features duplicated. |

**Selected:** Extend `notebook_data.ipynb`

**Follow-up — output files:**

| Option | Description |
|--------|-------------|
| New parquet files (feature_matrix_train/test.parquet) | Phase 1 parquet untouched; model notebooks load new files. |
| Overwrite Phase 1 parquet | Simpler load path, but Phase 1 output lost. |

**Selected:** New parquet files — `dados/feature_matrix_train.parquet` and `dados/feature_matrix_test.parquet`

---

### Rolling window scope

**Question:** For rolling goals/form (FEAT-01/02), should the 5-match window cross season boundaries?

| Option | Description |
|--------|-------------|
| Cross-season (continuous) | Round 1 of new season uses last 5 matches including end of previous season. Fewer NaN rows. |
| Season-scoped (reset per year) | Window resets each season. First 1-4 games per season have NaN. More realistic. |

**Selected:** Season-scoped — window resets at the start of each season

---

### NaN imputation strategy

**Question:** How should NaN values in rolling features be filled?

| Option | Description |
|--------|-------------|
| Fill with 0 | Simple, treats no-history as neutral. |
| Fill with per-team season mean | Two-pass computation, more informative. |
| Leave NaN for sklearn SimpleImputer | Pushes complexity to Phase 3/4. |
| (User custom) | Mean of last 10 matches from previous season |

**Selected:** Fill NaN with **mean of the team's last 10 matches from the previous season**

**Follow-up — fallback for teams with no previous season data:**

| Option | Description |
|--------|-------------|
| Fill with global season mean | Average of all teams in current season. |
| Fill with 0 | No-history baseline. |

**Selected:** Fill with 0 for teams with no prior season data

---

## Claude's Discretion

- Exact column naming for feature columns
- Whether FEAT-04 derived columns computed in `build_features` or post-processing
- Notebook cell ordering

## Deferred Ideas

- Head-to-head records (FEAT-V2-01) — v2 requirement
- Shot/possession stats from estatisticas CSV — v2
- API-Football live data — v2
