# Brazilian League Match Predictor

## What This Is

A Jupyter notebook-based ML system that predicts the outcome (Home Win / Draw / Away Win) of Brazilian football championship matches. Given two teams and their recent form — goals scored/conceded over their last 5 matches, win/draw/loss streak, and home vs. away records — the model classifies the most likely result. Data comes from the CSV files already in the repo supplemented by a live external API.

## Core Value

Given any two Brazilian league teams, predict the match outcome with meaningful accuracy — outperforming naive baselines and approaching the ~65-70% range typical of sports prediction models.

## Requirements

### Validated

- [x] Load and parse Brazilian league match data from `dados/archive/` CSVs — *Validated in Phase 1: Data Ingestion & Cleaning (2026-05-22)*

### Active
- [ ] Supplement historical data with live/current data from an external football API
- [ ] Engineer features: goals scored/conceded (last 5), W/D/L streak (last 5), home/away record
- [ ] Train a classifier to predict Home Win / Draw / Away Win
- [ ] Achieve ~65-70% accuracy on a held-out test set
- [ ] Deliver as a clean, well-documented Jupyter notebook

### Out of Scope

- Score/goal-range prediction — W/D/L classification is the target
- Web app or API — notebook-only for now
- Real-time betting integration — personal/academic project
- Leagues other than the Brazilian championship — out of scope for v1

## Context

- Repo already contains `dados/archive/` with Brazilian football championship CSVs (previously unused legacy data from an earlier project version)
- Current notebook (`notebook.ipynb`) handles MNIST classification — this is a separate, new deliverable
- Both an Insper AI trainee assignment and a personal interest project
- Stack: Python, Jupyter, Keras/TensorFlow (existing env), plus pandas/sklearn for feature engineering and data processing

## Constraints

- **Delivery format:** Jupyter notebook (consistent with trainee program format)
- **Data:** Must work with existing CSVs as baseline; API supplements for recency
- **Target accuracy:** ~65-70% on held-out test — competitive with sports prediction baselines
- **No hard grading constraints:** Free to choose architecture, no layer/size restrictions

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Notebook-first delivery | Consistent with Insper trainee format | — Pending |
| Seed from dados/archive/ + live API | Historical depth + current form | — Pending |
| W/D/L classification (not score prediction) | Simpler target, more data, clearer success metric | — Pending |

---
*Last updated: 2026-05-22 — Phase 1 complete (data pipeline + parquet outputs delivered)*

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state
