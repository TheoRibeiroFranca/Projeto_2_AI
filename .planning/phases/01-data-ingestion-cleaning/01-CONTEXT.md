# Phase 1: Data Ingestion & Cleaning - Context

**Gathered:** 2026-05-21
**Status:** Ready for planning

<domain>
## Phase Boundary

Load and clean the Brazilian league CSV data so downstream notebooks have a temporally-ordered `matches` DataFrame with canonical team names and a fixed train/test split boundary. The deliverable is a shared `notebook_data.ipynb` that saves cleaned parquet files. Feature engineering, model training, and evaluation are separate phases.

</domain>

<decisions>
## Implementation Decisions

### Notebook architecture
- **D-01:** Phase 1 produces `notebook_data.ipynb` — a dedicated data preparation notebook separate from the model notebooks.
- **D-02:** The notebook saves two parquet files: `dados/matches_train.parquet` and `dados/matches_test.parquet`. Model notebooks load these files directly rather than re-running data cleaning.

### Train/test split
- **D-03:** Split cutoff year is **2023**. Training data: 2003–2022 (~7,200 matches). Test data: 2023–2025 (~1,140 matches, ~3 seasons). Sort by date first, then apply the cutoff — never random shuffle.

### Target variable encoding
- **D-04:** Target column is named `result` and uses string labels: `'HomeWin'`, `'Draw'`, `'AwayWin'`. Derived from `vencedor` column (team name → HomeWin if equals mandante, `-` → Draw, team name → AwayWin if equals visitante). This column must be consistent across all 3 model notebooks.

### Team name normalization
- **D-05:** Build a manual canonical name dict before any `groupby` operation. The CSV team names are largely consistent already, but assert that top clubs (Flamengo, Corinthians, Sao Paulo, Fluminense, Santos, Internacional, Atletico-MG, Athletico-PR, Gremio, Palmeiras) each have 380+ match rows after normalization to confirm the dict is complete.

### Claude's Discretion
- Exact column renaming scheme for Portuguese → English (e.g., `mandante` → `home_team`, `visitante` → `away_team`)
- Whether to keep or drop columns not used downstream (arena, arrecadacao, formacao, tecnico)
- Date parsing format (`dayfirst=True` for DD/MM/YYYY)
- Whether to add a `season` column (year extracted from date) for season-level features later

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Data files
- `dados/archive/campeonato-brasileiro-full.csv` — Primary source. 9,165 matches (2003–2025). Columns in Portuguese: `mandante` (home), `visitante` (away), `vencedor` (winner or '-'), `data` (DD/MM/YYYY), `mandante_Placar`, `visitante_Placar` (scores), `rodata` (round).
- `dados/archive/campeonato-brasileiro-estatisticas-full.csv` — Shot/possession stats. Zero-filled pre-2013. NOT used in Phase 1 (deferred to v2).

### Project requirements
- `.planning/REQUIREMENTS.md` — DAT-01, DAT-02, DAT-03 are the Phase 1 requirements. Check acceptance criteria before planning tasks.
- `CLAUDE.md` — Critical gotchas section: temporal leakage rules, team name normalization, evaluation requirements. Read before planning any data task.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `notebook.ipynb` — Existing MNIST notebook. Not reused directly, but establishes the notebook style convention (Portuguese variable names for labels, matplotlib with `MPLBACKEND=agg` for headless rendering).

### Established Patterns
- Mixed Portuguese/English naming: data domain labels in Portuguese (`mandante`, `visitante`) may be kept internally; new computed columns in English (`home_team`, `away_team`, `result`).
- UV-managed dependencies: add `pandas` and `pyarrow` via `uv add pandas pyarrow` — they are listed in `CLAUDE.md` as needed for Phase 1.
- No automated formatter — manual PEP 8.

### Integration Points
- Output `dados/matches_train.parquet` + `dados/matches_test.parquet` are the contract boundary between Phase 1 and Phases 2–4. Column names and `result` encoding established here must be preserved by all downstream notebooks.

</code_context>

<specifics>
## Specific Ideas

- No specific references — open to standard pandas idioms for CSV loading and parquet saving.

</specifics>

<deferred>
## Deferred Ideas

- API-Football live data enrichment — v2 requirement (DAT-V2-01), not Phase 1.
- Shot/possession stats from `campeonato-brasileiro-estatisticas-full.csv` — v2 (pre-2013 data is zero-filled and contaminates rolling averages).

</deferred>

---

*Phase: 01-data-ingestion-cleaning*
*Context gathered: 2026-05-21*
