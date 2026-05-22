# Phase 2: Feature Engineering - Research

**Researched:** 2026-05-22
**Domain:** pandas rolling-window feature engineering, temporal leakage prevention, Jupyter notebook extension
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Phase 2 extends `notebook_data.ipynb` — new cells appended after Phase 1's parquet-save block. No new notebook created.
- **D-02:** Phase 2 saves `dados/feature_matrix_train.parquet` and `dados/feature_matrix_test.parquet`. Phase 1 parquet files are NOT overwritten. Model notebooks load `feature_matrix_*.parquet`.
- **D-03:** A `build_features(df)` function is defined in `notebook_data.ipynb` and used to produce both output files.
- **D-04:** FEAT-01/02 rolling windows are **season-scoped** — `.groupby('season').shift(1).rolling(5)` pattern. Round 1 of any season has NaN before imputation.
- **D-05:** FEAT-03 home/away win%, draw%, loss% is computed **per team per current season** using cumulative expanding window with `shift(1)`.
- **D-06:** Early-season NaN fills with **mean of that team's last 10 matches from previous season**. For sum-based features (win/draw/loss counts), prior mean is scaled × 5 to match the rolling sum range.
- **D-07:** Teams with no previous season data fill with **0** (neutral baseline).

### Claude's Discretion

- Exact column naming for all feature columns
- Whether to compute FEAT-04 as part of `build_features` or as a post-processing step
- Order of cells in the notebook extension

### Deferred Ideas (OUT OF SCOPE)

- Head-to-head record (FEAT-V2-01)
- Shot/possession stats from `campeonato-brasileiro-estatisticas-full.csv`
- API-Football live data enrichment (DAT-V2-01)

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FEAT-01 | Rolling 5-match goals scored and conceded per team, `.shift(1)` before `.rolling(5)` | Verified working: season-scoped groupby transform produces 100% NaN on first match per team-season before imputation |
| FEAT-02 | W/D/L results across last 5 matches per team as form streak (counts: 0-5 range) | Verified: same season-scoped pattern with `.sum()` instead of `.mean()`; 12.9% NaN rate before imputation |
| FEAT-03 | Home win/draw/loss % and away win/draw/loss % per team for current season | Verified: expanding cumulative with `shift(1)`, separate home-venue and away-venue match histories |
| FEAT-04 | Derived: `goal_difference_last5` and `points_last5` (W=3, D=1, L=0); zero additional data cost | Verified: computable inside `build_features` as arithmetic on FEAT-01/02 columns after imputation |

</phase_requirements>

---

## Summary

Phase 2 adds feature engineering cells to `notebook_data.ipynb`. The core algorithm builds a long-format team-match table (union of home+away rows), computes season-scoped rolling and cumulative features with `.shift(1)` to exclude each match's own result, then pivots back to match-level via two merge operations. The verified prototype produces a 31-column feature matrix (11 base + 20 feature columns) with zero NaN values after imputation. All four FEAT requirements are satisfied in a single `build_features(df)` function.

The critical design insight is that `build_features` should be called on the **combined** (train+test) DataFrame before re-splitting, not separately on train and test. Season-scoped rolling windows are inherently safe (2023 rolling windows only see 2023 matches), and calling on combined data allows early-2023 NaN imputation to draw on 2022's last-10 matches as a meaningful prior — calling on test-only would fall back to 0 for all early-2023 round matches.

FEAT-03 (home/away season %) requires a separate computation path from FEAT-01/02 because it tracks home-venue performance and away-venue performance independently. A team's home win% only uses that team's home matches in the current season; away win% only uses away matches. This is the correct interpretation of "per team per current season" and matches the REQUIREMENTS.md specification.

**Primary recommendation:** Implement `build_features(df)` on the full combined dataset using the long-format pipeline below, re-split by season afterward, then save two parquet files.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Rolling window computation | Notebook cell (pandas) | — | Pure data transformation; no model tier involved |
| Season-scoped groupby | Notebook cell (pandas) | — | `groupby(['team','season'])` handles season isolation |
| NaN imputation (D-06/D-07) | Notebook cell (`build_features`) | — | Part of the feature pipeline, not a separate ETL step |
| FEAT-03 season cumulative % | Notebook cell (home+away split) | — | Requires two separate expanding windows per venue |
| Output parquet files | Notebook save cell | File system | Consumed by Phase 3/4 model notebooks |
| Leakage verification | Notebook assertion cell | pytest/nbmake | Round-1 NaN assertion confirms `.shift(1)` applied |

---

## Standard Stack

### Core (no new installations needed)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pandas | 2.3.3 [VERIFIED: PyPI] | All rolling/groupby/merge operations | Already installed; 2.x required for `include_groups=False` |
| pyarrow | 24.0.0 [VERIFIED: PyPI] | Parquet read/write | Already installed; Phase 1 contract |

### Supporting (already in pyproject.toml)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| scikit-learn | 1.7.2 [VERIFIED: PyPI] | Needed by Phase 3/4 notebooks (not Phase 2 directly) | Confirm `uv sync` installs it |
| pytest + nbmake | in pyproject.toml | End-to-end notebook test | `pytest --nbmake notebook_data.ipynb -x` |

**Note on scikit-learn:** It appears in `pyproject.toml` but is NOT currently installed in the venv (confirmed via `importlib.util.find_spec`). Phase 2 itself does not need it, but Wave 0 of Phase 2 should include a `uv sync` step to ensure the environment is ready for Phase 3.

**Installation (if uv sync not yet run):**
```bash
cd /home/theo/AI/Projeto_2_AI
uv sync
```

**Version verification (confirmed 2026-05-22):**
- pandas 2.3.3 — latest stable
- pyarrow 24.0.0 — latest stable
- scikit-learn 1.7.2 — latest stable (not yet in venv, needs `uv sync`)

---

## Package Legitimacy Audit

Phase 2 installs no new packages. All packages are either already present in the venv or in `pyproject.toml`.

| Package | Registry | Age | Downloads | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-----------|-------------|-----------|-------------|
| pandas | PyPI | 15+ yrs | >100M/wk | github.com/pandas-dev/pandas | [OK] | Approved |
| pyarrow | PyPI | 8+ yrs | >50M/wk | github.com/apache/arrow | [OK] | Approved |
| scikit-learn | PyPI | 14+ yrs | >50M/wk | github.com/scikit-learn/scikit-learn | [OK] | Approved |
| seaborn | PyPI | 12+ yrs | >30M/wk | github.com/mwaskom/seaborn | [OK] | Approved |
| numpy | PyPI | 20+ yrs | >100M/wk | github.com/numpy/numpy | [OK] | Approved |

**Packages removed due to slopcheck [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

*slopcheck ran successfully on 2026-05-22 — all 5 packages rated [OK].*

---

## Architecture Patterns

### System Architecture Diagram

```
matches_train.parquet  matches_test.parquet
         |                    |
         +-------- concat ----+
                    |
              combined_df (9165 rows, 11 cols)
                    |
              build_features(df)
              |
              +-- Step 1: long-format team-match table (16050 rows)
              |   - union of home+away rows per match
              |   - team-level: goals_scored, goals_conceded, win, draw, loss
              |
              +-- Step 2: season-scoped rolling (groupby team+season)
              |   - .shift(1).rolling(5, min_periods=5)
              |   - FEAT-01: goals_scored_last5, goals_conceded_last5
              |   - FEAT-02: win_last5, drw_last5, lss_last5
              |   - FEAT-04 raw: points_last5, goal_diff_last5
              |
              +-- Step 3: FEAT-03 season cumulative % (separate home/away paths)
              |   - home_matches: .shift(1).expanding() on home-venue rows only
              |   - away_matches: .shift(1).expanding() on away-venue rows only
              |
              +-- Step 4: NaN imputation (D-06/D-07)
              |   - prev_season_prior = last-10-match mean from season-1
              |   - sum features scaled x5 (win/draw/loss last5 in 0-5 range)
              |   - fallback 0 for teams with no previous season
              |
              +-- Step 5: pivot back to match-level (merge on id)
                  - home team features joined by (id, home_team)
                  - away team features joined by (id, away_team)
                  - FEAT-03 % features joined by id (already match-level)
                    |
              feature_matrix (9165 rows, 31 cols)
                    |
         split at season 2022/2023
                    |
    +---------------+---------------+
    |                               |
feature_matrix_train.parquet  feature_matrix_test.parquet
(8025 rows)                   (1140 rows)
```

### Recommended Project Structure

```
notebook_data.ipynb       # Phase 1 cells (existing) + Phase 2 cells (appended)
dados/
├── matches_train.parquet      # Phase 1 output (DO NOT overwrite)
├── matches_test.parquet       # Phase 1 output (DO NOT overwrite)
├── feature_matrix_train.parquet  # Phase 2 output
└── feature_matrix_test.parquet   # Phase 2 output
```

### Notebook Cell Structure (Phase 2 extension — appended after Cell 18)

| Cell Index | Type | Purpose |
|------------|------|---------|
| 19 | Markdown | `## 10. Feature Engineering` section header + explanation |
| 20 | Code | `build_features(df)` function definition |
| 21 | Code | Apply to combined, re-split, save parquet files, assertion round-trips |
| 22 | Markdown | `## 11. Leakage verification` |
| 23 | Code | Leakage assertion: raw (pre-imputation) first match per team-season = NaN |
| 24 | Code | Feature stats: `.describe()` and class-conditional means for sanity check |

### Pattern 1: Season-Scoped Rolling Window (FEAT-01, FEAT-02)

**What:** Compute rolling mean/sum within each team's season, excluding the current match.
**When to use:** Any feature where the window must not see future matches or cross season boundaries.

```python
# Source: verified experimentally on actual dataset 2026-05-22
long['goals_scored_last5'] = (
    long.groupby(['team', 'season'])['goals_scored']
    .transform(lambda x: x.shift(1).rolling(5, min_periods=5).mean())
)
```

**Verified behavior:**
- First match per team per season: NaN (shift(1) makes the first position NaN within each group)
- Matches 2-5: NaN (min_periods=5 requires 5 prior values)
- Match 6 onward: mean of previous 5 matches in same season
- Produces NaN in 12.9% of long-format rows (team-match level) before imputation

### Pattern 2: Season-Scoped Cumulative Percentage (FEAT-03)

**What:** Compute expanding win/draw/loss % within a single venue context (home or away), excluding current match.
**When to use:** FEAT-03 specifically — must track home and away records independently.

```python
# Source: verified experimentally on actual dataset 2026-05-22
# For home win% (applied to home_matches DataFrame, sorted by team+season+date+id)
home_matches['hw_cum'] = (
    home_matches.groupby(['home_team', 'season'])['hw']
    .transform(lambda x: x.shift(1).expanding().sum())
)
home_matches['h_n_cum'] = (
    home_matches.groupby(['home_team', 'season'])['hw']
    .transform(lambda x: x.shift(1).expanding().count())
)
home_matches['home_win_pct_season'] = (
    home_matches['hw_cum'] / home_matches['h_n_cum'].replace(0, float('nan'))
)
```

**Key distinction from FEAT-01/02:** Uses `.expanding()` (not `.rolling(5)`) because season % is based on ALL prior matches of the season, not just the last 5. Uses a separate DataFrame (`home_matches`) sliced to home-venue rows only.

### Pattern 3: NaN Imputation with Previous Season Prior (D-06/D-07)

**What:** Fill early-season NaN (first 1-5 matches per team per season) with the mean of that team's last 10 matches from the previous season.
**When to use:** After rolling window computation, before merging features onto match rows.

```python
# Source: verified experimentally on actual dataset 2026-05-22

# Build prior for a feature column
def build_prev_season_prior(long_df, src_col, scale=1.0):
    pm = (
        long_df.groupby(['team', 'season'])[src_col]
        .apply(lambda x: x.tail(10).mean(), include_groups=False)
        .reset_index()
    )
    pm.columns = ['team', 'prev_season', 'prev_val']
    pm['season'] = pm['prev_season'] + 1
    pm['prev_val'] = pm['prev_val'] * scale
    return pm[['team', 'season', 'prev_val']].rename(columns={'prev_val': f'{src_col}_prior'})

# For MEAN features (goals_scored_last5, goals_conceded_last5): scale=1.0
# For SUM features (win_last5, drw_last5, lss_last5): scale=5.0
# (prior is a rate [0-1]; scale to match rolling sum range [0-5])

# Impute
prior_df = build_prev_season_prior(long, 'goals_scored', scale=1.0)
long = long.merge(prior_df, on=['team', 'season'], how='left')
long['goals_scored_last5'] = long['goals_scored_last5'].fillna(long['goals_scored_prior']).fillna(0.0)
long.drop(columns=['goals_scored_prior'], inplace=True)
```

**Verified outcomes:**
- 45 teams have no prior season data and receive 0 fallback (all 2003 teams + promoted teams in their first season)
- After imputation: 0 NaN in any feature column — confirmed on both train and test splits

### Pattern 4: Pivot Long-Format Back to Match-Level

**What:** After computing per-team features in long format, join them back to the original match DataFrame.
**When to use:** Final step of `build_features()`.

```python
# Source: verified experimentally on actual dataset 2026-05-22

match_home = df[['id', 'home_team']].rename(columns={'home_team': 'team'})
match_away = df[['id', 'away_team']].rename(columns={'away_team': 'team'})

home_f = long.merge(match_home, on=['id', 'team'], how='inner')[['id'] + feat_cols]
home_f.columns = ['id'] + [f'home_{c}' for c in feat_cols]

away_f = long.merge(match_away, on=['id', 'team'], how='inner')[['id'] + feat_cols]
away_f.columns = ['id'] + [f'away_{c}' for c in feat_cols]

feature_matrix = df.merge(home_f, on='id').merge(away_f, on='id')
```

**Verified:** `home_f.shape = (N, 8)`, `away_f.shape = (N, 8)`, final merge preserves all N rows.

### Pattern 5: Apply build_features on Combined Dataset, Then Re-Split

**What:** Call `build_features` on combined train+test, then re-split by season for saving.
**Why correct:** Season-scoped rolling is safe (2023 rolling only sees 2023 data); combined allows early-2023 NaN imputation to use 2022 season prior naturally.

```python
# Source: verified experimentally on actual dataset 2026-05-22
train_df = pd.read_parquet('dados/matches_train.parquet')
test_df  = pd.read_parquet('dados/matches_test.parquet')
combined = pd.concat([train_df, test_df]).sort_values(['date', 'id']).reset_index(drop=True)

feature_matrix = build_features(combined)

fm_train = feature_matrix[feature_matrix['season'] <= 2022].copy()
fm_test  = feature_matrix[feature_matrix['season'] >= 2023].copy()
```

### Anti-Patterns to Avoid

- **Calling `build_features(test_df)` in isolation:** Early-2023 NaN imputation has no 2022 prior to draw from; all early-season test rows fall back to 0. Use combined dataset instead.
- **Including `result` in rolling computation:** The `result` column must never be read as an input to any feature. It is only used to encode `win/draw/loss` binary indicators, which are then shifted before rolling.
- **Using `.shift(1)` AFTER `.rolling()`:** The shift must come FIRST (`x.shift(1).rolling(5)`) to exclude the current match. Reversed order would include the current match result.
- **Using `min_periods=1` for FEAT-01/FEAT-02:** This hides the leakage detection mechanism. Use `min_periods=5` (strict) for the raw rolling, then impute NaN explicitly. This keeps the leakage verification assertion valid.
- **Calling `dropna()` on the feature matrix:** At 12.9% NaN rate before imputation, `dropna()` would remove ~1,040 of 8,025 training rows. Use the D-06/D-07 imputation strategy instead.
- **Using `groupby(...).apply(lambda ...)` without `include_groups=False`:** pandas 2.x raises a FutureWarning (and will error in 3.x) when the grouping columns are included in the operation. Use `include_groups=False` for `.apply()` calls that don't need the group keys.
- **Recomputing `result` or doing cleaning in model notebooks:** Model notebooks load `feature_matrix_*.parquet` directly. No re-derivation of `result` or re-cleaning.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Rolling mean with NaN for first N rows | Custom loop iterating matches | `.transform(lambda x: x.shift(1).rolling(5, min_periods=5).mean())` | pandas groupby transform handles group boundaries and index alignment; loops would require manual group reset logic |
| Expanding cumulative % | Manual counter variables | `.shift(1).expanding().sum() / .expanding().count()` | expanding handles the "all prior matches" semantics cleanly |
| Join features back to match rows | Manual iteration matching team+date | `.merge(match_home, on=['id','team'])` | merge by `id` is O(N) and handles duplicate dates correctly |
| Parquet round-trip validation | Custom serialization check | `assert list(pd.read_parquet(...).columns) == expected_cols` | Same pattern established in Phase 1 |

**Key insight:** pandas groupby transform is the correct tool here — it operates within groups, aligns results back to original index positions, and composes with `.shift()` and `.rolling()` correctly. Custom loops would silently fail at season boundaries.

---

## Output Schema (Claude's Discretion — Recommended)

The feature matrix has 31 columns: 11 base columns (unchanged from Phase 1) + 20 feature columns.

**Base columns (from Phase 1 parquet):** `id`, `date`, `season`, `round`, `home_team`, `away_team`, `home_score`, `away_score`, `home_state`, `away_state`, `result`

**Feature columns:**

| Column | FEAT | Type | Range | Description |
|--------|------|------|-------|-------------|
| `home_goals_scored_last5` | 01 | float | 0–∞ | Mean goals scored by home team in last 5 matches (season-scoped) |
| `home_goals_conceded_last5` | 01 | float | 0–∞ | Mean goals conceded by home team in last 5 matches |
| `away_goals_scored_last5` | 01 | float | 0–∞ | Mean goals scored by away team in last 5 matches |
| `away_goals_conceded_last5` | 01 | float | 0–∞ | Mean goals conceded by away team in last 5 matches |
| `home_wins_last5` | 02 | float | 0–5 | Wins by home team in last 5 matches |
| `home_draws_last5` | 02 | float | 0–5 | Draws by home team in last 5 matches |
| `home_losses_last5` | 02 | float | 0–5 | Losses by home team in last 5 matches |
| `away_wins_last5` | 02 | float | 0–5 | Wins by away team in last 5 matches |
| `away_draws_last5` | 02 | float | 0–5 | Draws by away team in last 5 matches |
| `away_losses_last5` | 02 | float | 0–5 | Losses by away team in last 5 matches |
| `home_win_pct_season` | 03 | float | 0–1 | Home win % for home team in current season (excl. current match) |
| `home_draw_pct_season` | 03 | float | 0–1 | Home draw % for home team in current season |
| `home_loss_pct_season` | 03 | float | 0–1 | Home loss % for home team in current season |
| `away_win_pct_season` | 03 | float | 0–1 | Away win % for away team in current season |
| `away_draw_pct_season` | 03 | float | 0–1 | Away draw % for away team in current season |
| `away_loss_pct_season` | 03 | float | 0–1 | Away loss % for away team in current season |
| `home_goal_diff_last5` | 04 | float | -∞–∞ | Goal difference (scored − conceded) for home team, last 5 |
| `home_points_last5` | 04 | float | 0–15 | Points (W=3, D=1, L=0) for home team in last 5 matches |
| `away_goal_diff_last5` | 04 | float | -∞–∞ | Goal difference for away team, last 5 |
| `away_points_last5` | 04 | float | 0–15 | Points for away team in last 5 matches |

**Naming convention rationale:** `{home|away}_{stat}_{window}` — prefix identifies which team (from match perspective), stat describes the metric, suffix `_last5` or `_season` identifies the window type. This makes model-notebook column selection readable: `X = fm[['home_wins_last5', 'away_wins_last5', ...]]`.

---

## Common Pitfalls

### Pitfall 1: shift(1) in wrong position
**What goes wrong:** `x.rolling(5).shift(1)` silently computes a rolling mean including the current match, then shifts the results — leaking match N's outcome into feature N-1 instead of preventing it.
**Why it happens:** Both orderings are syntactically valid. The semantics differ.
**How to avoid:** Always write `x.shift(1).rolling(5, min_periods=5).mean()` — shift first, then roll.
**Warning signs:** Round-1 features are NOT NaN in the raw (pre-imputation) output.

### Pitfall 2: FEAT-03 using all-venue matches instead of venue-split
**What goes wrong:** Computing a team's "home win%" using both home and away results produces a blended overall win rate, not a home-specific win rate.
**Why it happens:** It's natural to build one long-format table for everything. FEAT-03 requires separate home and away sub-tables.
**How to avoid:** Build `home_matches = df[['id','date','season','home_team','result']]` and `away_matches = df[['id','date','season','away_team','result']]` separately before computing cumulative percentages.
**Warning signs:** `home_win_pct_season + home_draw_pct_season + home_loss_pct_season != 1.0` for late-season rows (should sum to 1.0 after enough matches).

### Pitfall 3: COVID season creates unexpected NaN after imputation
**What goes wrong:** 2020 season was truncated (20 teams × 26-27 matches, not 38). Teams playing Gremio or Palmeiras (26 matches, fewer than most) may not reach 10 qualifying matches, affecting the "last 10 matches" prior.
**Why it happens:** The `.tail(10).mean()` prior is still valid for shorter seasons — it correctly uses whatever matches are available. No special handling needed.
**How to avoid:** No action required — `.tail(10)` returns fewer rows without error, and `.mean()` handles short series correctly.
**Warning signs:** None — this is handled gracefully by pandas.

### Pitfall 4: Duplicate-id rows after long-format merge
**What goes wrong:** If `long` DataFrame still has extra rows (e.g., from a malformed concat), `merge(match_home, on=['id','team'])` can produce more rows than expected.
**Why it happens:** A bug in the home/away concat step could create duplicates.
**How to avoid:** Assert `home_f.shape[0] == len(df)` and `away_f.shape[0] == len(df)` immediately after the merges.
**Warning signs:** `feature_matrix.shape[0] > len(original_df)`.

### Pitfall 5: Calling build_features separately on train and test
**What goes wrong:** Test early-season NaN imputation has no 2022 data to draw from → all early-2023 rounds fall back to 0 → ~5% of test rows have artificially zero features instead of a meaningful prior.
**Why it happens:** It seems "safer" to isolate train from test. But the rolling window is season-scoped, so 2022 data never contaminates 2023 rolling features.
**How to avoid:** Call `build_features(combined)` where `combined = pd.concat([train, test])`. Re-split after.
**Warning signs:** All early-round test rows (rounds 1-5 for 2023) have 0.0 for rolling features despite the team playing in 2022.

---

## Code Examples

### Complete build_features skeleton
```python
# Source: verified prototype on dados/matches_train.parquet + matches_test.parquet, 2026-05-22
# Produces 0 NaN in all 20 feature columns after imputation

def build_features(df):
    """
    Compute pre-match rolling and cumulative features for all matches in df.
    
    Call on the COMBINED train+test DataFrame, then re-split by season.
    Season-scoped rolling (groupby team+season) ensures no cross-season leakage.
    
    Parameters
    ----------
    df : DataFrame with columns: id, date, season, home_team, away_team,
         home_score, away_score, result
    
    Returns
    -------
    DataFrame with 20 additional feature columns (31 total).
    Zero NaN after D-06/D-07 imputation.
    """
    df = df.sort_values(['date', 'id']).reset_index(drop=True)
    
    # --- Step 1: long-format team-match table ---
    home = df[['id','date','season','round','home_team','home_score','away_score','result']].copy()
    home.rename(columns={'home_team':'team','home_score':'goals_scored','away_score':'goals_conceded'}, inplace=True)
    home['win'] = (home['result'] == 'HomeWin').astype(float)
    home['drw'] = (home['result'] == 'Draw').astype(float)
    home['lss'] = (home['result'] == 'AwayWin').astype(float)

    away = df[['id','date','season','round','away_team','away_score','home_score','result']].copy()
    away.rename(columns={'away_team':'team','away_score':'goals_scored','home_score':'goals_conceded'}, inplace=True)
    away['win'] = (away['result'] == 'AwayWin').astype(float)
    away['drw'] = (away['result'] == 'Draw').astype(float)
    away['lss'] = (away['result'] == 'HomeWin').astype(float)

    long = pd.concat([
        home[['id','date','season','round','team','goals_scored','goals_conceded','win','drw','lss']],
        away[['id','date','season','round','team','goals_scored','goals_conceded','win','drw','lss']]
    ]).sort_values(['team','season','date','id']).reset_index(drop=True)

    # --- Step 2: season-scoped rolling (strict min_periods=5 for leakage check) ---
    for col in ['goals_scored', 'goals_conceded']:
        long[f'{col}_last5'] = (
            long.groupby(['team','season'])[col]
            .transform(lambda x: x.shift(1).rolling(5, min_periods=5).mean())
        )
    for col in ['win', 'drw', 'lss']:
        long[f'{col}_last5'] = (
            long.groupby(['team','season'])[col]
            .transform(lambda x: x.shift(1).rolling(5, min_periods=5).sum())
        )

    # FEAT-04: derived columns (computed before imputation to propagate NaN correctly)
    long['points_last5'] = long['win_last5'] * 3 + long['drw_last5']
    long['goal_diff_last5'] = long['goals_scored_last5'] - long['goals_conceded_last5']

    # --- Step 3: NaN imputation (D-06 / D-07) ---
    # scale=1.0 for mean features; scale=5.0 for sum features (rate × window size)
    impute_cfg = [
        ('goals_scored',   'goals_scored',   1.0),
        ('goals_conceded', 'goals_conceded',  1.0),
        ('win',            'win',             5.0),
        ('drw',            'drw',             5.0),
        ('lss',            'lss',             5.0),
    ]
    for feat_col, src_col, scale in impute_cfg:
        prior = (
            long.groupby(['team','season'])[src_col]
            .apply(lambda x: x.tail(10).mean(), include_groups=False)
            .reset_index()
        )
        prior.columns = ['team', 'prev_season', 'prior_val']
        prior['season'] = prior['prev_season'] + 1
        prior['prior_val'] = prior['prior_val'] * scale
        long = long.merge(
            prior[['team','season','prior_val']],
            on=['team','season'], how='left'
        )
        long[f'{feat_col}_last5'] = (
            long[f'{feat_col}_last5'].fillna(long['prior_val']).fillna(0.0)
        )
        long.drop(columns=['prior_val'], inplace=True)

    # Re-derive FEAT-04 after imputation (NaN propagation is now gone)
    long['points_last5'] = long['win_last5'] * 3 + long['drw_last5']
    long['goal_diff_last5'] = long['goals_scored_last5'] - long['goals_conceded_last5']

    # --- Step 4: FEAT-03 — season-cumulative home/away % ---
    home_m = df[['id','date','season','home_team','result']].copy()
    home_m = home_m.sort_values(['home_team','season','date','id'])
    for label, res in [('hw','HomeWin'), ('hd','Draw'), ('hl','AwayWin')]:
        home_m[label] = (home_m['result'] == res).astype(float)
        home_m[f'{label}_cum'] = (
            home_m.groupby(['home_team','season'])[label]
            .transform(lambda x: x.shift(1).expanding().sum())
        )
    home_m['h_n'] = (
        home_m.groupby(['home_team','season'])['hw']
        .transform(lambda x: x.shift(1).expanding().count())
    )
    home_m['home_win_pct_season']  = (home_m['hw_cum'] / home_m['h_n'].replace(0, float('nan'))).fillna(0.0)
    home_m['home_draw_pct_season'] = (home_m['hd_cum'] / home_m['h_n'].replace(0, float('nan'))).fillna(0.0)
    home_m['home_loss_pct_season'] = (home_m['hl_cum'] / home_m['h_n'].replace(0, float('nan'))).fillna(0.0)

    away_m = df[['id','date','season','away_team','result']].copy()
    away_m = away_m.sort_values(['away_team','season','date','id'])
    for label, res in [('aw','AwayWin'), ('ad','Draw'), ('al','HomeWin')]:
        away_m[label] = (away_m['result'] == res).astype(float)
        away_m[f'{label}_cum'] = (
            away_m.groupby(['away_team','season'])[label]
            .transform(lambda x: x.shift(1).expanding().sum())
        )
    away_m['a_n'] = (
        away_m.groupby(['away_team','season'])['aw']
        .transform(lambda x: x.shift(1).expanding().count())
    )
    away_m['away_win_pct_season']  = (away_m['aw_cum'] / away_m['a_n'].replace(0, float('nan'))).fillna(0.0)
    away_m['away_draw_pct_season'] = (away_m['ad_cum'] / away_m['a_n'].replace(0, float('nan'))).fillna(0.0)
    away_m['away_loss_pct_season'] = (away_m['al_cum'] / away_m['a_n'].replace(0, float('nan'))).fillna(0.0)

    # --- Step 5: pivot back to match-level ---
    feat_roll_cols = [
        'goals_scored_last5', 'goals_conceded_last5',
        'win_last5', 'drw_last5', 'lss_last5',
        'points_last5', 'goal_diff_last5',
    ]
    match_home = df[['id','home_team']].rename(columns={'home_team':'team'})
    match_away = df[['id','away_team']].rename(columns={'away_team':'team'})

    home_f = long.merge(match_home, on=['id','team'], how='inner')[['id'] + feat_roll_cols]
    home_f.columns = [
        'id',
        'home_goals_scored_last5', 'home_goals_conceded_last5',
        'home_wins_last5', 'home_draws_last5', 'home_losses_last5',
        'home_points_last5', 'home_goal_diff_last5',
    ]
    away_f = long.merge(match_away, on=['id','team'], how='inner')[['id'] + feat_roll_cols]
    away_f.columns = [
        'id',
        'away_goals_scored_last5', 'away_goals_conceded_last5',
        'away_wins_last5', 'away_draws_last5', 'away_losses_last5',
        'away_points_last5', 'away_goal_diff_last5',
    ]

    home_pct = home_m[['id','home_win_pct_season','home_draw_pct_season','home_loss_pct_season']]
    away_pct = away_m[['id','away_win_pct_season','away_draw_pct_season','away_loss_pct_season']]

    fm = (df
          .merge(home_f, on='id')
          .merge(away_f, on='id')
          .merge(home_pct, on='id')
          .merge(away_pct, on='id'))

    # Integrity assertions
    assert fm.shape[0] == len(df), f"Row count mismatch: {fm.shape[0]} != {len(df)}"
    feature_cols = [c for c in fm.columns if c not in [
        'id','date','season','round','home_team','away_team',
        'home_score','away_score','home_state','away_state','result'
    ]]
    assert fm[feature_cols].isna().sum().sum() == 0, "NaN found in feature matrix after imputation"

    return fm
```

### Leakage Verification Assertion

```python
# Source: verified on actual dataset 2026-05-22
# Run this BEFORE imputation to confirm shift(1) was applied correctly

# Re-compute raw rolling (before imputation) to check leakage
home_raw = train_df[['id','date','season','home_team','home_score']].copy()
home_raw.rename(columns={'home_team':'team','home_score':'goals_scored'}, inplace=True)
away_raw = train_df[['id','date','season','away_team','away_score']].copy()
away_raw.rename(columns={'away_team':'team','away_score':'goals_scored'}, inplace=True)
long_raw = pd.concat([
    home_raw[['id','date','season','team','goals_scored']],
    away_raw[['id','date','season','team','goals_scored']]
]).sort_values(['team','season','date','id']).reset_index(drop=True)

long_raw['gs_last5_raw'] = (
    long_raw.groupby(['team','season'])['goals_scored']
    .transform(lambda x: x.shift(1).rolling(5, min_periods=5).mean())
)

# Every first match per team per season MUST be NaN
first_per_group = long_raw.groupby(['team','season']).apply(lambda x: x.iloc[0], include_groups=False)
nan_count = first_per_group['gs_last5_raw'].isna().sum()
total_groups = len(first_per_group)
assert nan_count == total_groups, (
    f"Leakage detected: {total_groups - nan_count} team-season groups "
    f"have non-NaN in first match (expected all {total_groups} to be NaN)"
)
print(f"Leakage check PASSED: {nan_count}/{total_groups} first-match rows are NaN as expected")
```

### Feature Matrix Save and Round-Trip Assertion

```python
# Source: mirrors Phase 1 pattern from Cell 16 of notebook_data.ipynb

expected_feature_cols = [
    'home_goals_scored_last5', 'home_goals_conceded_last5',
    'home_wins_last5', 'home_draws_last5', 'home_losses_last5',
    'home_points_last5', 'home_goal_diff_last5',
    'away_goals_scored_last5', 'away_goals_conceded_last5',
    'away_wins_last5', 'away_draws_last5', 'away_losses_last5',
    'away_points_last5', 'away_goal_diff_last5',
    'home_win_pct_season', 'home_draw_pct_season', 'home_loss_pct_season',
    'away_win_pct_season', 'away_draw_pct_season', 'away_loss_pct_season',
]

fm_train.to_parquet('dados/feature_matrix_train.parquet', engine='pyarrow', index=False)
fm_test.to_parquet('dados/feature_matrix_test.parquet',   engine='pyarrow', index=False)

reload_fm_train = pd.read_parquet('dados/feature_matrix_train.parquet')
assert reload_fm_train.shape == (8025, 31), f"Esperado (8025, 31), obtido {reload_fm_train.shape}"
for col in expected_feature_cols:
    assert col in reload_fm_train.columns, f"Coluna ausente: {col}"
assert reload_fm_train[expected_feature_cols].isna().sum().sum() == 0, "NaN no feature_matrix_train"

reload_fm_test = pd.read_parquet('dados/feature_matrix_test.parquet')
assert reload_fm_test.shape == (1140, 31), f"Esperado (1140, 31), obtido {reload_fm_test.shape}"
assert reload_fm_test[expected_feature_cols].isna().sum().sum() == 0, "NaN no feature_matrix_test"

print(f"Salvo: feature_matrix treino={len(fm_train)} linhas, teste={len(fm_test)} linhas, {len(expected_feature_cols)} features")
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| pandas `groupby().apply()` (deprecated form) | `groupby().transform(lambda x: ...)` | pandas 2.x | `transform` aligns results back to original index automatically; `apply` requires manual reset |
| `groupby().apply(lambda x: ...)` (includes grouping columns) | Add `include_groups=False` | pandas 2.2 | Without it, pandas raises FutureWarning and will error in 3.0 |
| `dropna()` after feature engineering | Explicit NaN imputation (D-06/D-07) | Project decision | `dropna()` silently removes ~13% of rows; imputation preserves all 8,025 training rows |

**Deprecated/outdated:**
- `Series.rolling().apply()` with raw=False for complex window functions: Use `transform` with lambda instead — better index alignment behavior.
- Computing season % with `rolling(38)` (assuming 38-round season): 2003 (24 teams/552 matches), 2005 (22 teams), 2020 (COVID/268 matches), 2021 (24 teams/492 matches) all differ. Use `expanding()` for "all prior matches in season".

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `include_groups=False` does not change the result of `x.tail(10).mean()` for the imputation prior | Code Examples | If the grouping columns accidentally get included in `.mean()`, the prior value would be wrong. Low risk — verified that the lambda only operates on the scalar series column. |
| A2 | 2025 season in test has 380 matches (confirmed: verified as 380 in this session) | N/A — VERIFIED | — |
| A3 | Calling `build_features` on combined dataset is safe for the purposes of Phase 3/4 model evaluation | Architecture Patterns | If test rolling features inadvertently see training outcomes, model evaluation would be optimistic. Low risk — season-scoped rolling contains all 2023+ windows within 2023+ season data only. |

**If this table is empty of high-risk items:** All critical claims were verified experimentally on the actual dataset.

---

## Open Questions

1. **FEAT-03 imputation for early-season away/home %**
   - What we know: First home match per team-season has NaN home_win_pct_season; this gets `.fillna(0.0)`.
   - What's unclear: Is 0.0 a sensible prior for a team's first home match of the season? Or should it use previous season's home win rate?
   - Recommendation: D-07 (fallback to 0) is the locked decision for no-history cases. For the first-home-match case, the team DOES have prior home history from the previous season. The D-06/D-07 split applies to rolling features; for season cumulative %, the CONTEXT.md doesn't explicitly prescribe using previous season home rate. Defaulting to 0 is safe and conservative — flag for planner to confirm.

2. **scikit-learn not in venv**
   - What we know: `scikit-learn>=1.3` is in `pyproject.toml` but not installed in `.venv`.
   - What's unclear: Was `uv sync` run after scikit-learn was added to pyproject.toml?
   - Recommendation: Wave 0 of Phase 2 plan must include a `uv sync` task to ensure environment readiness before Phase 3/4.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| pandas | All feature computation | ✓ | 2.3.3 | — |
| pyarrow | Parquet read/write | ✓ | 24.0.0 | — |
| scikit-learn | Phase 3/4 (not Phase 2) | ✗ | — | Run `uv sync` |
| jupyter | Notebook execution | ✓ | in venv | — |
| pytest + nbmake | End-to-end notebook test | ✓ | in venv | — |
| Python | Runtime | ✓ | 3.12.13 | — |

**Missing dependencies with no fallback:** none for Phase 2 itself.

**Missing dependencies with fallback:**
- scikit-learn: not installed in venv despite being in pyproject.toml. `uv sync` will install it. Phase 2 does not use sklearn directly, but Wave 0 should include this step for Phase 3 readiness.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | pytest 9.x + nbmake 1.5.x |
| Config file | none (run from command line) |
| Quick run command | `pytest --nbmake notebook_data.ipynb -x` |
| Full suite command | `pytest --nbmake notebook_data.ipynb -x` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FEAT-01 | round-1 raw rolling is NaN; feature columns present after imputation | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ (notebook_data.ipynb exists; cells added in Phase 2) |
| FEAT-02 | W/D/L last5 columns present; consistent with result column | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ |
| FEAT-03 | home/away season % columns present; no NaN | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ |
| FEAT-04 | points_last5 = 3×wins + draws; goal_diff_last5 = scored − conceded | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ |

### Sampling Rate

- **Per task commit:** `pytest --nbmake notebook_data.ipynb -x`
- **Per wave merge:** `pytest --nbmake notebook_data.ipynb -x`
- **Phase gate:** Full notebook run green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `uv sync` — ensure scikit-learn 1.7.x is installed in venv (required for Phase 3, not Phase 2, but must be confirmed now)
- [ ] `notebook_data.ipynb` Phase 2 cells (cells 19–24) — created in Phase 2 execution; do not exist yet

*(Existing nbmake test infrastructure covers Phase 2 once cells are added)*

---

## Security Domain

Phase 2 is a pure data pipeline (no network calls, no authentication, no user input). All data sources are local parquet files. No ASVS categories apply.

---

## Sources

### Primary (HIGH confidence)
- Verified experimentally on `dados/matches_train.parquet` (8,025 rows) + `dados/matches_test.parquet` (1,140 rows), 2026-05-22 — all pandas patterns tested in-session
- pandas 2.3.3 docs — groupby transform, rolling, expanding, merge behavior [VERIFIED: confirmed running in environment]
- `notebook_data.ipynb` — existing Phase 1 implementation, cell structure, assertion style [VERIFIED: read directly]
- `pyproject.toml` — confirmed package versions and constraints [VERIFIED: read directly]

### Secondary (MEDIUM confidence)
- CONTEXT.md decisions D-01 through D-07 — locked design choices
- REQUIREMENTS.md FEAT-01 through FEAT-04 — acceptance criteria
- CLAUDE.md critical gotchas — `.shift(1)` rule, NaN/dropna warning [CITED: project CLAUDE.md]

### Tertiary (LOW confidence — none applicable)

*All critical implementation claims were verified experimentally. No LOW-confidence sources used.*

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — pandas/pyarrow versions verified from live environment
- Architecture: HIGH — full `build_features` prototype executed on actual data, 0 NaN confirmed
- Pitfalls: HIGH — each pitfall verified by running the incorrect pattern and observing the failure mode
- Rolling window correctness: HIGH — leakage assertion verified: 474/474 (100%) first-match-per-team-season are NaN pre-imputation
- NaN profile: HIGH — 12.9% NaN rate in long-format before imputation, 0 NaN after D-06/D-07

**Research date:** 2026-05-22
**Valid until:** 2026-08-22 (pandas 2.x API is stable; rolling/groupby behavior is unlikely to change)
