# Phase 2: Feature Engineering - Context

**Gathered:** 2026-05-22
**Status:** Ready for planning

<domain>
## Phase Boundary

Extend `notebook_data.ipynb` with feature engineering cells that produce a `feature_matrix` DataFrame — every row is a match with pre-match-only rolling-window features and no data leakage. The deliverable is two new parquet files consumed by all three model notebooks. Model training, evaluation, and documentation are separate phases.

</domain>

<decisions>
## Implementation Decisions

### Notebook architecture
- **D-01:** Phase 2 extends `notebook_data.ipynb` — new cells are appended after Phase 1's parquet-save block. No new notebook is created; the data + features pipeline lives in a single notebook.
- **D-02:** Phase 2 saves two new files: `dados/feature_matrix_train.parquet` and `dados/feature_matrix_test.parquet`. Phase 1 parquet files (`matches_train.parquet`, `matches_test.parquet`) are NOT overwritten. Model notebooks load `feature_matrix_*.parquet`.
- **D-03:** A `build_features(df)` function is defined in `notebook_data.ipynb` and used to produce both output files, so the feature matrix can be reconstructed from any filtered date range without modifying the pipeline.

### Rolling window scope
- **D-04:** FEAT-01/02 rolling windows (goals scored, goals conceded, form streak) are **season-scoped** — the `.groupby('season').shift(1).rolling(5)` pattern ensures the window resets at the start of each season. Round 1 of any season has NaN (no prior matches in that season yet).
- **D-05:** FEAT-03 (home win%, draw%, loss%, away win%, draw%, loss%) is computed **per team per current season** — explicitly season-scoped per REQUIREMENTS.md.

### NaN imputation strategy
- **D-06:** For early-season matches with NaN rolling features (first 1-4 games per team per season): fill with the **mean of that team's last 10 matches from the previous season**. This provides a meaningful prior rather than a hard zero.
- **D-07:** Fallback when a team has no previous season data (first-ever appearance, promoted team with no prior CSV history): fill with **0**. Treats no-history as a neutral baseline.

### Claude's Discretion
- Exact column naming for all feature columns (e.g., `home_goals_scored_last5`, `away_form_wins_last5`)
- Whether to compute FEAT-04 (goal_difference_last5, points_last5) as part of `build_features` or as a post-processing step
- Order of cells in the notebook extension

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase 2 requirements
- `.planning/REQUIREMENTS.md` — FEAT-01 through FEAT-04 (feature requirements with exact acceptance criteria). Success criterion 5 requires `build_features(df)` to be reusable.
- `CLAUDE.md` — Critical gotchas section: `.shift(1)` before `.rolling()` rule, draw class collapse, NaN handling warning (`dropna()` drops ~20% of rows).

### Phase 1 output contract (Phase 2 inputs)
- `.planning/phases/01-data-ingestion-cleaning/01-CONTEXT.md` — Column names (`home_team`, `away_team`, `season`, `round`, `result`, `home_score`, `away_score`), `result` encoding, parquet file paths.
- `dados/matches_train.parquet` — 8,025 rows (2003–2022), 11 columns. Phase 2 reads this as input.
- `dados/matches_test.parquet` — 1,140 rows (2023–2025). Phase 2 reads this as input.

### Roadmap
- `.planning/ROADMAP.md` — Phase 2 success criteria (5 items): NaN check on round-1 rows, presence of all feature columns, season-scoped home/away %, derived columns, `build_features` reusability.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `notebook_data.ipynb` — Phase 1 notebook to be extended. Already loads and cleans CSV, normalizes team names, applies temporal split, and saves Phase 1 parquet. Phase 2 cells are appended after the final save block.
- `season` column — Phase 1 already derives a `season` column (year from date). This is the groupby key for season-scoped rolling windows.

### Established Patterns
- `.shift(1)` before `.rolling()` — locked pattern from CLAUDE.md; prevents same-match leakage in rolling features.
- Temporal split via `season` — Phase 1 split at 2023 cutoff. Phase 2 must apply `build_features()` to both train and test DataFrames separately to avoid any test→train leakage in rolling windows.
- `result` column: `'HomeWin'` / `'Draw'` / `'AwayWin'` — must not be used as input to any feature computation.

### Integration Points
- Output files `dados/feature_matrix_train.parquet` + `dados/feature_matrix_test.parquet` are the Phase 2 → Phase 3/4 contract. All column names established here must be stable across the three model notebooks.
- `build_features(df)` is defined in `notebook_data.ipynb`; model notebooks load from parquet (they do not call this function directly).

</code_context>

<specifics>
## Specific Ideas

- NaN fill strategy: use the mean of that team's last 10 matches from the previous season (not a global mean or 0, except as fallback for truly new teams).

</specifics>

<deferred>
## Deferred Ideas

- Head-to-head record for the two specific teams (FEAT-V2-01) — v2 requirement, not Phase 2.
- Shot/possession stats from `campeonato-brasileiro-estatisticas-full.csv` — v2 (pre-2013 data is zero-filled).
- API-Football live data enrichment (DAT-V2-01) — v2.

</deferred>

---

*Phase: 02-feature-engineering*
*Context gathered: 2026-05-22*
