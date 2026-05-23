---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: ready_to_execute
stopped_at: Phase 3 planned — ready to execute
last_updated: "2026-05-23T14:00:00.000Z"
progress:
  total_phases: 4
  completed_phases: 2
  total_plans: 5
  completed_plans: 3
  percent: 50
---

# Project State: Brazilian League Match Predictor

*This file is project memory. It is updated at every phase transition and plan completion.*

---

## Project Reference

**Core Value:** Given any two Brazilian league teams, predict the match outcome (W/D/L) with ~65-70% accuracy using the last 5 matches of form data

**Current Focus:** Phase 3 — baseline model notebooks

---

## Current Position

| Field | Value |
|-------|-------|
| Milestone | v1 |
| Current Phase | 3 — Baseline Model Notebooks |
| Phase Status | Planned — ready to execute (2 plans, Wave 1) |
| Overall Progress | 2 / 4 phases complete |

**Progress bar:** `████░░░░░░` 50%

---

## Performance Metrics

| Metric | Target | Current |
|--------|--------|---------|
| Overall accuracy (held-out test) | ~65-70% | TBD |
| Naive baseline (always Home Win) | ~49.6% | **49.6%** (corrected — ~46% in CLAUDE.md was wrong; verified from dataset) |
| Draw recall | > 0% | TBD |
| Macro-F1 | Competitive with accuracy | TBD |

---

## Phase Completion Log

| Phase | Name | Plans | Date | Key Output |
|-------|------|-------|------|------------|
| 01 | Data Ingestion & Cleaning | 1/1 | 2026-05-22 | notebook_data.ipynb, matches_train.parquet (8025 rows), matches_test.parquet (1140 rows) |

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
| HomeWin baseline = 49.6% (not 46%) | Dataset verification shows 49.6%; old ~46% figure was incorrect | Phase 1 |
| canonical_names dict is empty (no active variants) | 46 unique team names, all internally consistent; dict is defensive guard | Phase 1 |

### Critical Pitfalls to Avoid

- **Random train/test split** — always sort by date and split at fixed year cutoff
- **Missing `.shift(1)`** — verify round-1 rows are NaN after feature engineering
- **Draw class collapse** — always use `class_weight='balanced'` and report `classification_report`
- **`dropna()` on feature matrix** — impute NaN instead; `dropna()` can drop ~20% of rows

### Open Questions

1. Does API-Football free tier cover Brasileirao (league ID 71)? (v2 concern; not blocking v1)
2. ~~What is the actual home win rate in this dataset?~~ RESOLVED: **49.6%** (Phase 1 EDA)
3. ~~How many team name variants exist?~~ RESOLVED: **46 unique names, 0 variants** (Phase 1 normalization)
4. Where exactly does the stats CSV have reliable non-zero data? (Audit in Phase 1 if shot/possession features are considered for v2)

### Blockers

*(None currently)*

### Todos

- [x] Run `uv add scikit-learn pandas seaborn` — DONE (seaborn, pytest, nbmake added in Phase 1)
- [x] Verify `campeonato-brasileiro-full.csv` schema — DONE (17 columns verified, 9165 rows confirmed)
- [ ] Run `/gsd:verify-work` for Phase 1 before proceeding to Phase 2

---

## Session Continuity

**Last active session:** 2026-05-22 (Phase 1 execution)
**Stopped at:** Phase 3 context gathered
**Resume point:** Run `/gsd:verify-work` for Phase 1, then proceed to Phase 2

---

*State initialized: 2026-05-21*
*Last updated: 2026-05-22*
