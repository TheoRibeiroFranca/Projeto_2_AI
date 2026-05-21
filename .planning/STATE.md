---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: unknown
last_updated: "2026-05-21T20:48:14.651Z"
progress:
  total_phases: 4
  completed_phases: 0
  total_plans: 0
  completed_plans: 0
  percent: 0
---

# Project State: Brazilian League Match Predictor

*This file is project memory. It is updated at every phase transition and plan completion.*

---

## Project Reference

**Core Value:** Given any two Brazilian league teams, predict the match outcome (W/D/L) with ~65-70% accuracy using the last 5 matches of form data

**Current Focus:** Phase 1 — Data Ingestion & Cleaning

---

## Current Position

| Field | Value |
|-------|-------|
| Milestone | v1 |
| Current Phase | 1 — Data Ingestion & Cleaning |
| Current Plan | None (not started) |
| Phase Status | Not started |
| Overall Progress | 0 / 4 phases complete |

**Progress bar:** `░░░░░░░░░░` 0%

---

## Performance Metrics

| Metric | Target | Current |
|--------|--------|---------|
| Overall accuracy (held-out test) | ~65-70% | TBD |
| Naive baseline (always Home Win) | ~46% | TBD (verify in Phase 1 EDA) |
| Draw recall | > 0% | TBD |
| Macro-F1 | Competitive with accuracy | TBD |

---

## Phase Completion Log

*(Empty — no phases complete yet)*

---

## Accumulated Context

### Key Decisions

| Decision | Rationale | Phase |
|----------|-----------|-------|
| sklearn ensembles over Keras/TF | Wrong tool for tabular data; RF/GBM dominate | Pre-planning |
| Notebook-per-model delivery | Consistent with Insper trainee format; separate notebooks for logistic, RF, GBM | Pre-planning |
| Temporal split only — no KFold | Data has time dependency; random splits cause leakage | Pre-planning |
| `class_weight='balanced'` on all models | Draws are ~26% of data; naive training collapses draw recall | Pre-planning |
| `.shift(1)` before `.rolling(5)` | Prevents same-match data leakage in rolling window features | Pre-planning |
| Macro-F1 as primary metric | More informative than accuracy with class imbalance | Pre-planning |

### Critical Pitfalls to Avoid

- **Random train/test split** — always sort by date and split at fixed year cutoff
- **Missing `.shift(1)`** — verify round-1 rows are NaN after feature engineering
- **Draw class collapse** — always use `class_weight='balanced'` and report `classification_report`
- **`dropna()` on feature matrix** — impute NaN instead; `dropna()` can drop ~20% of rows

### Open Questions

1. Does API-Football free tier cover Brasileirao (league ID 71)? (v2 concern; not blocking v1)
2. What is the actual home win rate in this dataset? (Verify in Phase 1 EDA)
3. How many team name variants exist? (Verify with `df['mandante'].nunique()` in Phase 1)
4. Where exactly does the stats CSV have reliable non-zero data? (Audit in Phase 1 if shot/possession features are considered for v2)

### Blockers

*(None currently)*

### Todos

- [ ] Run `uv add scikit-learn pandas seaborn` to add required packages
- [ ] Verify `campeonato-brasileiro-full.csv` schema matches research assumptions (columns: `partida_id`, `mandante`, `visitante`, `data`, `mandante_placar`, `visitante_placar`)

---

## Session Continuity

**Last active session:** 2026-05-21 (initialization)
**Resume point:** Start Phase 1 — run `/gsd:plan-phase 1`

---

*State initialized: 2026-05-21*
*Last updated: 2026-05-21*
