# Phase 1: Data Ingestion & Cleaning - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions captured in CONTEXT.md — this log preserves the discussion.

**Date:** 2026-05-21
**Phase:** 01-data-ingestion-cleaning
**Mode:** discuss (default)
**Areas discussed:** Notebook architecture, Train/test cutoff year, Target variable encoding

---

## Areas Selected

User selected all three presented gray areas.

---

## Discussion Log

### Notebook architecture

**Options presented:**
- Shared data notebook (saves parquet/CSV) — `notebook_data.ipynb` saves files, model notebooks load them
- Inline per-model notebook — each model notebook contains its own data loading code (duplicated 3×)

**User selected:** Shared data notebook (saves parquet/CSV)

**Follow-up: Output format**

**Options presented:**
- Parquet — preserves dtypes, fast, small (recommended)
- CSV — human-readable but loses dtype info

**User selected:** Parquet

**Decision captured:** `notebook_data.ipynb` saves `dados/matches_train.parquet` + `dados/matches_test.parquet`.

---

### Train/test cutoff year

**Options presented:**
- 2023 — Train: 2003–2022 (~7,200 matches), Test: 2023–2025 (~1,140 matches) [recommended]
- 2024 — Train: 2003–2023 (~7,580 matches), Test: 2024–2025 (~760 matches)
- 2022 — Train: 2003–2021 (~6,825 matches), Test: 2022–2025 (~1,520 matches)

**User selected:** 2023 (recommended)

**Decision captured:** Cutoff = year < 2023 for train, year >= 2023 for test.

---

### Target variable encoding

**Options presented:**
- 'H' / 'D' / 'A' strings — standard sports analytics convention [recommended]
- 'HomeWin' / 'Draw' / 'AwayWin' — more explicit and self-documenting

**User selected:** 'HomeWin' / 'Draw' / 'AwayWin' strings

**Decision captured:** `result` column uses `'HomeWin'`, `'Draw'`, `'AwayWin'` strings derived from `vencedor` column.

---

## No Corrections / Deferred Ideas

No scope creep attempted. Discussion stayed within Phase 1 boundary.
