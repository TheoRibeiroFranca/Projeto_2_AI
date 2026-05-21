# Phase 1: Data Ingestion & Cleaning - Research

**Researched:** 2026-05-21
**Domain:** Pandas CSV ingestion, temporal train/test split, parquet serialization, Jupyter notebook EDA
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Phase 1 produces `notebook_data.ipynb` — a dedicated data preparation notebook separate from the model notebooks.
- **D-02:** The notebook saves two parquet files: `dados/matches_train.parquet` and `dados/matches_test.parquet`. Model notebooks load these files directly rather than re-running data cleaning.
- **D-03:** Split cutoff year is **2023**. Training data: 2003–2022 (~7,200 matches). Test data: 2023–2025 (~1,140 matches, ~3 seasons). Sort by date first, then apply the cutoff — never random shuffle.
- **D-04:** Target column is named `result` with string labels `'HomeWin'`, `'Draw'`, `'AwayWin'`. Derived from `vencedor` column (team name → HomeWin if equals mandante, `-` → Draw, team name → AwayWin if equals visitante). This column must be consistent across all 3 model notebooks.
- **D-05:** Build a manual canonical name dict before any `groupby` operation. Assert that top clubs (Flamengo, Corinthians, Sao Paulo, Fluminense, Santos, Internacional, Atletico-MG, Athletico-PR, Gremio, Palmeiras) each have 380+ match rows after normalization to confirm the dict is complete.

### Claude's Discretion

- Exact column renaming scheme for Portuguese → English (e.g., `mandante` → `home_team`, `visitante` → `away_team`)
- Whether to keep or drop columns not used downstream (arena, arrecadacao, formacao, tecnico)
- Date parsing format (`dayfirst=True` for DD/MM/YYYY)
- Whether to add a `season` column (year extracted from date) for season-level features later

### Deferred Ideas (OUT OF SCOPE)

- API-Football live data enrichment — v2 requirement (DAT-V2-01), not Phase 1.
- Shot/possession stats from `campeonato-brasileiro-estatisticas-full.csv` — v2 (pre-2013 data is zero-filled and contaminates rolling averages).
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DAT-01 | User can load and parse `campeonato-brasileiro-full.csv` as the primary historical data source (9,165 matches, 2003–2025) | VERIFIED: CSV loads cleanly with `pd.read_csv`; 9,165 rows confirmed; all 17 columns valid; 0 parse errors |
| DAT-02 | System normalizes team names to a canonical mapping before any grouping or feature engineering | VERIFIED: Only 46 unique team names exist across the full dataset; names are already internally consistent; no multi-variant collisions detected; canonical dict is a pass-through with no active remapping needed; 380+ assertion passes for all 10 top clubs in the train set |
| DAT-03 | System applies temporal train/test split — data sorted by date, fixed year cutoff, never random shuffle | VERIFIED: CSV is already sorted by date; `year <= 2022` cutoff yields 8,025 train rows and 1,140 test rows with zero overlap |
</phase_requirements>

---

## Summary

The primary data source (`dados/archive/campeonato-brasileiro-full.csv`) is in excellent condition: 9,165 rows, no duplicate match keys, zero null values in the columns needed downstream, no date parse errors, and team names that are already internally consistent across 22 seasons. The CSV is already sorted chronologically, so temporal ordering requires only verification, not resorting.

The most important finding for planning is that team name normalization is simpler than the CLAUDE.md "20+ years of naming variants" warning implies. Direct inspection shows only 46 unique team names with zero multi-variant collisions for the same club (e.g., no "Atletico MG" alongside "Atletico-MG"). The canonical name dict will be a defensive correctness measure rather than an active repair operation, but the assertion (`>= 380 rows per top club`) still matters as a regression guard for future data additions.

The `vencedor → result` derivation logic is clean: `-` is always a Draw; every other `vencedor` value exactly matches either `mandante` or `visitante` with 0 unclassifiable rows. The actual home win rate in this dataset is **49.6%**, not the ~46% mentioned in STATE.md — this matters for naive baseline comparisons reported in Phase 3 EDA.

**Primary recommendation:** The pipeline is `pd.read_csv → date parse → result derivation → column rename/select → year-based split → to_parquet`. No complex normalization, no imputation, no deduplication needed. The notebook will be straightforward — effort is mostly in correctness assertions and EDA quality.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| CSV ingestion | Notebook (single-process Python) | — | Local file; no server, no API |
| Date parsing & sort verification | Notebook | — | pandas in-memory operation |
| Team name normalization | Notebook | — | Static dict, applied before any groupby |
| `result` target encoding | Notebook | — | Row-wise derivation from existing columns |
| Train/test split | Notebook | — | Year-cutoff applied to in-memory DataFrame |
| Parquet serialization | Notebook → `dados/` filesystem | — | Contract boundary for downstream phases |
| EDA & class distribution | Notebook (markdown + output cells) | — | Visual verification artifact |

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pandas | 2.3.3 (already installed) | CSV reading, DataFrame manipulation, parquet write | Only data manipulation library; already in `pyproject.toml` |
| pyarrow | 24.0.0 | Parquet engine for `df.to_parquet()` | Required by pandas for parquet; official Apache Arrow project |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| seaborn | 0.13.2 | Class distribution bar chart in EDA cell | Cleaner bar charts than raw matplotlib for categorical counts |
| matplotlib | 3.8+ (already installed) | Fallback plotting; headless backend via `MPLBACKEND=agg` | Already in project; required for headless Jupyter rendering |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| pyarrow (parquet) | fastparquet | Both are valid; pyarrow is the pandas-recommended default and more actively maintained |
| `pd.to_datetime(format='%d/%m/%Y')` | `read_csv(parse_dates=..., dayfirst=True)` | Explicit `pd.to_datetime` after load is clearer and more explicit in notebooks; avoids infer_datetime_format deprecation warning in pandas 2.x |
| seaborn barplot | `df['result'].value_counts().plot(kind='bar')` | Both work; seaborn produces publication-quality output with less code |

**Installation (to add to project):**
```bash
uv add pyarrow seaborn
```

---

## Package Legitimacy Audit

> Packages to install: `pyarrow`, `seaborn` (pandas already present).

| Package | Registry | Age | Downloads | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-----------|-------------|-----------|-------------|
| pyarrow | PyPI | ~8 yrs | Very high (Apache Arrow project) | github.com/apache/arrow | [OK] | Approved |
| pandas | PyPI | ~15 yrs | Very high | github.com/pandas-dev/pandas | [OK] | Approved (already installed) |
| seaborn | PyPI | ~12 yrs | Very high | github.com/mwaskom/seaborn | [OK] | Approved |

**Packages removed due to slopcheck [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

[VERIFIED: npm registry] — slopcheck 0.6.1 ran successfully and returned `[OK]` for all three packages against PyPI.

**Version verification (PyPI):**
```bash
pip index versions pyarrow   # → 24.0.0 (latest)
pip index versions pandas    # → 2.3.3 (already installed, matches)
pip index versions seaborn   # → 0.13.2 (latest)
```
All versions confirmed against PyPI registry on 2026-05-21. [VERIFIED: PyPI registry]

---

## Architecture Patterns

### System Architecture Diagram

```
dados/archive/campeonato-brasileiro-full.csv
        │
        ▼
[Cell 1: Setup]
  os.environ MPLBACKEND=agg
  imports: pandas, seaborn, matplotlib
        │
        ▼
[Cell 2: Load CSV]
  pd.read_csv(...) → raw_df (9165 rows × 17 cols)
  assert len == 9165
        │
        ▼
[Cell 3: Date Parse + Sort Verify]
  pd.to_datetime(format='%d/%m/%Y') → df['date']
  assert df['date'].is_monotonic_increasing
        │
        ▼
[Cell 4: Derive result column]
  vencedor == '-'         → 'Draw'
  vencedor == mandante    → 'HomeWin'
  vencedor == visitante   → 'AwayWin'
  assert 0 unclassifiable rows
        │
        ▼
[Cell 5: Team Name Normalization]
  canonical_names = { ... }  ← manual dict
  apply to home_team, away_team
  assert top 10 clubs >= 380 rows
        │
        ▼
[Cell 6: Column Rename + Select]
  rename: mandante→home_team, visitante→away_team,
          data→date, mandante_Placar→home_score,
          visitante_Placar→away_score, rodata→round
  drop: arena, arrecadacao, formacao_*, tecnico_*
  optional: add season = date.dt.year
        │
        ▼
[Cell 7: Temporal Split]
  train = df[df['date'].dt.year <= 2022]   (8025 rows)
  test  = df[df['date'].dt.year >= 2023]   (1140 rows)
  assert len(train) + len(test) == 9165
  assert no index overlap
        │
        ├──────────────────────┐
        ▼                      ▼
[Cell 8: Save Parquet]    [Cell 9: EDA]
  dados/matches_train.parquet  class distribution bar chart
  dados/matches_test.parquet   home win rate print
  assert files exist           assert HomeWin ≈ 49.6%
```

### Recommended Project Structure

```
dados/
├── archive/
│   └── campeonato-brasileiro-full.csv   # source (read-only)
├── matches_train.parquet                # Phase 1 output
└── matches_test.parquet                 # Phase 1 output
notebook_data.ipynb                      # Phase 1 deliverable
```

### Pattern 1: Pandas Date Parse (pandas 2.x compatible)

**What:** Parse DD/MM/YYYY date strings to datetime64 after CSV load, not inside `read_csv`.
**When to use:** Always — avoids the deprecated `infer_datetime_format` and `date_parser` kwargs in pandas 2.x.
**Example:**
```python
# Source: pandas 2.x official docs (verified via pandas.__doc__)
df['date'] = pd.to_datetime(df['data'], format='%d/%m/%Y')
# Verify sort order — CSV is pre-sorted but assert defensively
assert df['date'].is_monotonic_increasing, "CSV is not sorted by date — sort required"
```

### Pattern 2: Derive `result` from `vencedor`

**What:** Row-wise derivation of target variable from existing winner column.
**When to use:** Once, immediately after load.
**Example:**
```python
def derive_result(row):
    if row['vencedor'] == '-':
        return 'Draw'
    elif row['vencedor'] == row['mandante']:
        return 'HomeWin'
    else:
        return 'AwayWin'

df['result'] = df.apply(derive_result, axis=1)
# Verify completeness
assert df['result'].isnull().sum() == 0
assert set(df['result'].unique()) == {'HomeWin', 'Draw', 'AwayWin'}
```

### Pattern 3: Temporal Train/Test Split

**What:** Year-cutoff split — never shuffle, never use random_state.
**When to use:** The only acceptable split method for time-series data.
**Example:**
```python
train_df = df[df['date'].dt.year <= 2022].copy()
test_df  = df[df['date'].dt.year >= 2023].copy()
# Assertions for correctness
assert len(train_df) == 8025, f"Expected 8025 train rows, got {len(train_df)}"
assert len(test_df) == 1140, f"Expected 1140 test rows, got {len(test_df)}"
assert len(set(train_df.index) & set(test_df.index)) == 0, "Train/test index overlap!"
```

### Pattern 4: Parquet Save with pyarrow

**What:** Save cleaned DataFrames to parquet for downstream notebooks.
**When to use:** After all cleaning, normalization, and split — final step.
**Example:**
```python
import os
os.makedirs('dados', exist_ok=True)
train_df.to_parquet('dados/matches_train.parquet', engine='pyarrow', index=False)
test_df.to_parquet('dados/matches_test.parquet', engine='pyarrow', index=False)
# Verify files were written
assert os.path.exists('dados/matches_train.parquet')
assert os.path.exists('dados/matches_test.parquet')
```

### Pattern 5: Team Name Canonical Dict (defensive)

**What:** Apply a normalization dict even though the current CSV has no active variants. This guards against future data additions and makes the assertion meaningful.
**When to use:** Before any groupby or feature computation.
**Example:**
```python
# Names in the CSV are already consistent (46 unique names, 0 conflicts).
# Dict is a pass-through for all current data — it exists for correctness guarantee.
canonical_names = {
    # Current data has no variants; add entries here if variants appear
    # e.g., 'Atletico MG': 'Atletico-MG'  (not currently present)
}

for col in ['home_team', 'away_team']:
    df[col] = df[col].replace(canonical_names)

# Assert top clubs meet 380+ row threshold (counts both home and away)
top_clubs = ['Flamengo', 'Corinthians', 'Sao Paulo', 'Fluminense', 'Santos',
             'Internacional', 'Atletico-MG', 'Athletico-PR', 'Gremio', 'Palmeiras']
team_counts = pd.concat([df['home_team'], df['away_team']]).value_counts()
for club in top_clubs:
    assert team_counts.get(club, 0) >= 380, f"{club} has fewer than 380 rows: {team_counts.get(club, 0)}"
```

### Anti-Patterns to Avoid

- **`read_csv(parse_dates=['data'])`:** Deprecated `infer_datetime_format` path in pandas 2.x. Use `pd.to_datetime(format=...)` after load.
- **`df.sort_values('data')`:** Sorting the string `data` column instead of the parsed `date` column gives wrong order for day-first dates (DD/MM/YYYY sorts lexicographically wrong). Always parse first, then sort if needed.
- **`random_state` or `shuffle=True` in split:** Never shuffle time-series data. The year-cutoff is the only valid split.
- **`dropna()` on the full DataFrame:** Would silently drop 4,975 rows with empty `formacao_*` columns. Drop only unused columns; keep score and match identity columns which have 0 nulls.
- **Counting only `mandante` rows for the 380+ assertion:** Top clubs appear as both home and away. Assert against `pd.concat([df['home_team'], df['away_team']]).value_counts()`, not just one column.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Parquet file format | Custom CSV-with-metadata | `df.to_parquet(engine='pyarrow')` | Typed columns, ~10x smaller than CSV, preserves datetime64 dtype without re-parsing |
| Date parsing | Regex-based string splitting | `pd.to_datetime(format='%d/%m/%Y')` | Handles edge cases, leap years, vectorized |
| Class distribution chart | Manual `plt.bar` with count calculations | `seaborn.countplot` or `df['result'].value_counts().plot(kind='bar')` | One line, correct labeling |

**Key insight:** The data preparation pipeline is almost entirely standard pandas idioms. No custom code is needed beyond the `derive_result` function and the canonical name dict.

---

## Common Pitfalls

### Pitfall 1: Home Win Rate Discrepancy

**What goes wrong:** The naive baseline is stated as "~46%" in STATE.md and CLAUDE.md. Actual data shows **49.6%** home win rate. Reporting ~46% in the EDA cell is factually wrong and will create confusion when Phase 3 notebooks claim to beat a baseline that does not match the data.
**Why it happens:** The ~46% figure may come from a different dataset or global football averages.
**How to avoid:** Compute and assert the home win rate directly in the EDA cell using the loaded data. The correct figures are: HomeWin 49.6%, Draw 26.4%, AwayWin 23.9%.
**Warning signs:** Any stated naive baseline that does not match the value derived from `df['result'].value_counts(normalize=True)`.

### Pitfall 2: `dropna()` Removing Valid Rows

**What goes wrong:** `formacao_mandante`, `formacao_visitante` (4,975 nulls each), `tecnico_mandante`, `tecnico_visitante` (4,610 nulls each), and `arrecadacao` (8,785 nulls) have many missing values. A `df.dropna()` call would drop 8,785 rows (96% of the dataset).
**Why it happens:** Reflexive data cleaning without checking which columns are null.
**How to avoid:** Drop unused columns first, then verify the remaining columns have 0 nulls. The columns needed downstream (ID, rodata, data, mandante, visitante, vencedor, mandante_Placar, visitante_Placar, mandante_Estado, visitante_Estado) have **0 null values**.
**Warning signs:** DataFrame shape drops below 9,165 after any null-handling step.

### Pitfall 3: `arena` Column Leading `\xa0` Characters

**What goes wrong:** 5,077 arena rows have a leading non-breaking space (`\xa0`), e.g., `'\xa0Brinco de Ouro'`. If arena is kept, string comparisons fail silently.
**Why it happens:** Source data encoding artifact.
**How to avoid:** Drop `arena` column (it is not used downstream). If kept, apply `.str.strip()` before any use.
**Warning signs:** `df['arena'].str.startswith('\xa0').sum() > 0` returns True.

### Pitfall 4: pyarrow Not in Project venv

**What goes wrong:** `df.to_parquet(...)` fails with `ImportError: Missing optional dependency 'pyarrow'` because pyarrow is not in `pyproject.toml`. pandas requires either `pyarrow` or `fastparquet` as a parquet engine — neither is currently installed in the project venv.
**Why it happens:** pyarrow is a runtime dependency of `pd.to_parquet`, not of pandas itself. It is not auto-installed.
**How to avoid:** Run `uv add pyarrow` before implementing the parquet save cell. Add this as the first task in Wave 0.
**Warning signs:** `python -c "import pyarrow"` fails in `.venv`.

### Pitfall 5: Year-Based Split Assertion Values

**What goes wrong:** Asserting `len(train_df) == 7200` (from CONTEXT.md estimate) fails — actual count is **8,025**. The estimate was approximate.
**Why it happens:** 2003–2004 had 552 matches/year (expansion season); 2020 had only 268 (COVID disruption); 2021 had 492 (COVID overflow).
**How to avoid:** Assert exact counts: `train == 8025`, `test == 1140`. These are verified from the actual file.
**Warning signs:** Assertion fails with "Expected 7200, got 8025".

### Pitfall 6: 2020/2021 COVID Anomaly

**What goes wrong:** 2020 had only 268 matches (COVID suspension); 2021 had 492 (makeup season finishing in Dec 2021). If code assumes exactly 380 rows per season for any year-level validation, it will fail for these seasons.
**Why it happens:** Season was suspended and resumed mid-year, spilling into the next calendar year.
**How to avoid:** Do not write assertions that assume 380 rows per calendar year. The split by calendar year is still correct — 2021 matches belong to the 2021 CSV year regardless of the competition season.
**Warning signs:** Per-season row count assertions that expect exactly 380 rows/year.

---

## Code Examples

Verified patterns from official sources and direct data inspection:

### Full Pipeline Skeleton

```python
# Source: direct CSV inspection + pandas 2.x API (verified 2026-05-21)
import os
import pandas as pd
import matplotlib
os.environ['MPLBACKEND'] = 'agg'
import matplotlib.pyplot as plt
import seaborn as sns

# ── 1. Load ────────────────────────────────────────────────────────────────
df = pd.read_csv('dados/archive/campeonato-brasileiro-full.csv')
assert len(df) == 9165, f"Expected 9165 rows, got {len(df)}"
assert list(df.columns) == [
    'ID', 'rodata', 'data', 'hora', 'mandante', 'visitante',
    'formacao_mandante', 'formacao_visitante', 'tecnico_mandante',
    'tecnico_visitante', 'vencedor', 'arena', 'mandante_Placar',
    'visitante_Placar', 'mandante_Estado', 'visitante_Estado', 'arrecadacao'
]

# ── 2. Parse date ──────────────────────────────────────────────────────────
df['date'] = pd.to_datetime(df['data'], format='%d/%m/%Y')
assert df['date'].is_monotonic_increasing, "Data not sorted by date"

# ── 3. Derive target ───────────────────────────────────────────────────────
def derive_result(row):
    if row['vencedor'] == '-':
        return 'Draw'
    elif row['vencedor'] == row['mandante']:
        return 'HomeWin'
    else:
        return 'AwayWin'

df['result'] = df.apply(derive_result, axis=1)
assert df['result'].isnull().sum() == 0
assert set(df['result'].unique()) == {'HomeWin', 'Draw', 'AwayWin'}

# ── 4. Rename & select columns ─────────────────────────────────────────────
df = df.rename(columns={
    'mandante':        'home_team',
    'visitante':       'away_team',
    'mandante_Placar': 'home_score',
    'visitante_Placar':'away_score',
    'rodata':          'round',
    'mandante_Estado': 'home_state',
    'visitante_Estado':'away_state',
})
df['season'] = df['date'].dt.year  # optional: useful for Phase 2 season features

keep = ['ID', 'date', 'season', 'round', 'home_team', 'away_team',
        'home_score', 'away_score', 'home_state', 'away_state', 'result']
df = df[keep]

# ── 5. Team name normalization ─────────────────────────────────────────────
canonical_names = {}  # no active variants in current data
for col in ['home_team', 'away_team']:
    df[col] = df[col].replace(canonical_names)

top_clubs = ['Flamengo', 'Corinthians', 'Sao Paulo', 'Fluminense', 'Santos',
             'Internacional', 'Atletico-MG', 'Athletico-PR', 'Gremio', 'Palmeiras']
team_counts = pd.concat([df['home_team'], df['away_team']]).value_counts()
for club in top_clubs:
    assert team_counts.get(club, 0) >= 380, f"{club}: {team_counts.get(club, 0)} < 380"

# ── 6. Temporal split ──────────────────────────────────────────────────────
train_df = df[df['date'].dt.year <= 2022].copy()
test_df  = df[df['date'].dt.year >= 2023].copy()
assert len(train_df) == 8025
assert len(test_df)  == 1140
assert len(set(train_df.index) & set(test_df.index)) == 0

# ── 7. Save parquet ────────────────────────────────────────────────────────
os.makedirs('dados', exist_ok=True)
train_df.to_parquet('dados/matches_train.parquet', engine='pyarrow', index=False)
test_df.to_parquet('dados/matches_test.parquet',   engine='pyarrow', index=False)
assert os.path.exists('dados/matches_train.parquet')
assert os.path.exists('dados/matches_test.parquet')
print(f"Saved: train={len(train_df)} rows, test={len(test_df)} rows")
```

### EDA Cell: Class Distribution

```python
# Source: seaborn countplot + verified class counts (2026-05-21)
fig, ax = plt.subplots(figsize=(6, 4))
order = ['HomeWin', 'Draw', 'AwayWin']
sns.countplot(data=df, x='result', order=order, ax=ax)
ax.set_title('Match Result Distribution (2003–2025)')
ax.set_xlabel('Result')
ax.set_ylabel('Count')
for p in ax.patches:
    ax.annotate(f'{p.get_height()} ({p.get_height()/len(df)*100:.1f}%)',
                (p.get_x() + p.get_width()/2., p.get_height()),
                ha='center', va='bottom', fontsize=9)
plt.tight_layout()
plt.show()

# Print verified class rates (must match bar chart)
print("Class distribution:")
print(df['result'].value_counts())
print()
print(f"Naive 'always HomeWin' baseline accuracy: {(df['result']=='HomeWin').mean():.1%}")
# Expected output: ~49.6% (NOT 46% — this dataset has higher home advantage)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `read_csv(parse_dates=..., infer_datetime_format=True)` | `pd.to_datetime(format='...')` after load | pandas 2.0 (2023) | `infer_datetime_format` deprecated; explicit format is cleaner anyway |
| `read_csv(date_parser=...)` | `pd.to_datetime(format=...)` after load | pandas 2.0 (2023) | `date_parser` deprecated in pandas 2.0 |
| `fastparquet` as parquet engine | `pyarrow` | ~2020 | pyarrow is now the pandas-recommended default engine |

**Deprecated/outdated:**
- `infer_datetime_format=True` in `read_csv`: deprecated in pandas 2.0; raises warning in 2.3.x
- `date_parser=` kwarg in `read_csv`: deprecated in pandas 2.0

---

## Runtime State Inventory

> Phase 1 is greenfield (no rename/refactor). No runtime state migration required.

Section omitted per greenfield phase rules.

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | The `season` column (year from date) is useful for Phase 2 season-level features | Standard Stack / Code Examples | Low — it is cheap to add and easy to drop if not needed |
| A2 | `home_state` and `away_state` columns are worth keeping in the parquet output (they have 0 nulls and may be useful for analysis) | Standard Stack | Low — easy to drop in Phase 2 if not needed |

**Verified facts (not assumptions):**
- 9,165 rows — [VERIFIED: direct CSV inspection]
- 0 duplicate match keys — [VERIFIED: direct CSV inspection]
- All 46 team names already canonical — [VERIFIED: direct CSV inspection]
- CSV already sorted by date — [VERIFIED: direct CSV inspection with `is_monotonic_increasing`]
- Train split = 8,025 rows, test = 1,140 rows — [VERIFIED: direct CSV inspection]
- Home win rate = 49.6% — [VERIFIED: direct CSV inspection]
- pyarrow not in project venv — [VERIFIED: `python -c "import pyarrow"` fails in `.venv`]
- pyarrow 24.0.0 is current PyPI version — [VERIFIED: PyPI registry]

---

## Open Questions

1. **Column selection scope**
   - What we know: Downstream phases (Feature Engineering) need `home_team`, `away_team`, `date`, `home_score`, `away_score`, `round`, `season`, `result`.
   - What's unclear: Whether `home_state`, `away_state`, `ID` are needed in Phase 2/3/4.
   - Recommendation: Keep all non-dropped columns in the parquet. Storage cost is negligible (~10 columns, 9K rows). Downstream can ignore unneeded columns.

2. **Naive baseline value for Phase 3 comparison**
   - What we know: Home win rate in this dataset is **49.6%**, not ~46%.
   - What's unclear: Whether the ~46% figure in CLAUDE.md/STATE.md was intentional (e.g., adjusted for Draw class imbalance) or simply an error.
   - Recommendation: Use the empirically computed value (49.6%) in the EDA cell. Document the discrepancy in a markdown cell. The Phase 3 planner should use 49.6% as the naive baseline.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python 3.12 | Runtime | ✓ | 3.12.13 | — |
| pandas | CSV/parquet operations | ✓ | 2.3.3 (in .venv) | — |
| pyarrow | `df.to_parquet()` | ✗ (not in .venv) | — | Run `uv add pyarrow` (Wave 0 task) |
| seaborn | EDA countplot | ✗ (not in .venv) | — | Run `uv add seaborn` or use `df.plot(kind='bar')` |
| matplotlib | Headless plot rendering | ✓ | 3.8+ (in .venv) | — |
| jupyter | Notebook execution | ✓ | ipykernel 7.2.0 | — |
| jupyter nbconvert | Notebook CI run | ✓ | Bundled with jupyter | — |
| dados/ directory | Parquet output location | ✓ (exists) | — | `os.makedirs('dados', exist_ok=True)` |

**Missing dependencies with no fallback:**
- `pyarrow` — required for `to_parquet`; must be added via `uv add pyarrow` before the notebook can execute to completion.

**Missing dependencies with fallback:**
- `seaborn` — fallback to `df['result'].value_counts().plot(kind='bar')` using matplotlib alone.

---

## Validation Architecture

> nyquist_validation is enabled in .planning/config.json.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | pytest + nbmake (notebook execution test) |
| Config file | None — Wave 0 creates `pytest.ini` |
| Quick run command | `pytest --nbmake notebook_data.ipynb -x` |
| Full suite command | `pytest --nbmake notebook_data.ipynb` |

**Rationale for nbmake:** The Phase 1 deliverable is a notebook, not a library. The most meaningful test is "does the notebook run end-to-end without errors and produce the expected output files?" nbmake executes notebooks via pytest and treats cell execution errors as test failures. Individual assertion cells inside the notebook (assert len == 9165, assert no overlap, etc.) serve as in-notebook unit tests.

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DAT-01 | CSV loads without error; 9,165 rows confirmed | Smoke (notebook execution) | `pytest --nbmake notebook_data.ipynb -x` | ❌ Wave 0 |
| DAT-02 | Team name normalization; 380+ rows per top club asserted | Unit (inline assert in notebook) | `pytest --nbmake notebook_data.ipynb -x` | ❌ Wave 0 |
| DAT-03 | Temporal split produces 8,025 train / 1,140 test; no overlap asserted | Unit (inline assert in notebook) | `pytest --nbmake notebook_data.ipynb -x` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** Not applicable (single notebook deliverable)
- **Per wave merge:** `pytest --nbmake notebook_data.ipynb`
- **Phase gate:** Notebook runs end-to-end; both parquet files exist; EDA cell renders; all inline assertions pass.

### Wave 0 Gaps

- [ ] `notebook_data.ipynb` — does not exist yet; created in Wave 0/1
- [ ] `pytest.ini` — no pytest config present; Wave 0 creates minimal config
- [ ] `uv add pyarrow seaborn pytest nbmake` — required before any notebook execution

---

## Security Domain

> This phase handles only local CSV files and writes local parquet files. No network calls, no authentication, no user input, no secrets.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Local notebook; no auth |
| V3 Session Management | No | No sessions |
| V4 Access Control | No | Local filesystem only |
| V5 Input Validation | Partial | Assert row count and result uniqueness after load |
| V6 Cryptography | No | No encryption needed |

**Minimal security surface:** The only external input is the CSV file already checked into the repo. The only output is local parquet files. No user-supplied data, no network calls, no secrets.

---

## Sources

### Primary (HIGH confidence — direct data inspection)

- `dados/archive/campeonato-brasileiro-full.csv` — row count, column names, dtypes, null counts, date range, team names, class distribution, sort order all verified by direct Python inspection on 2026-05-21
- `dados/archive/Legenda.txt` — official column definitions for Portuguese → English mapping
- `.planning/phases/01-data-ingestion-cleaning/01-CONTEXT.md` — locked decisions D-01 through D-05
- pandas 2.3.3 `__doc__` — API signature for `read_csv`, `to_datetime`, `to_parquet` verified in project venv

### Secondary (HIGH confidence — registry verification)

- PyPI registry — `pip index versions` for pyarrow (24.0.0), pandas (2.3.3), seaborn (0.13.2) — verified 2026-05-21
- slopcheck 0.6.1 — `[OK]` verdict for pyarrow, pandas, seaborn — verified 2026-05-21

### Tertiary (ASSUMED — not verified this session)

- None

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all packages verified on PyPI; installed in system Python and confirmed available
- Architecture: HIGH — based on direct file inspection; all row counts, dtypes, and split values computed from actual data
- Pitfalls: HIGH — arrecadacao null count (8785), arena whitespace (\xa0), COVID year row counts, and home win rate all confirmed from direct data inspection
- Validation architecture: MEDIUM — nbmake approach is standard for notebook testing but requires installing nbmake + pytest which are not yet in the project venv

**Research date:** 2026-05-21
**Valid until:** 2027-05-21 (data is static; no moving targets in this phase)
