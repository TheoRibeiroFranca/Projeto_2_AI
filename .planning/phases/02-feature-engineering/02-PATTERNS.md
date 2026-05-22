# Phase 2: Feature Engineering - Pattern Map

**Mapped:** 2026-05-22
**Files analyzed:** 3 (1 notebook extended, 2 new parquet outputs)
**Analogs found:** 3 / 3

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `notebook_data.ipynb` (cells 19–24) | notebook/pipeline | batch transform | `notebook_data.ipynb` cells 14–16 (temporal split + parquet save) | exact — same notebook, same coding conventions |
| `dados/feature_matrix_train.parquet` | data output | batch | `dados/matches_train.parquet` | exact — same save/assert pattern |
| `dados/feature_matrix_test.parquet` | data output | batch | `dados/matches_test.parquet` | exact — same save/assert pattern |

---

## Pattern Assignments

### New cells 19–24 in `notebook_data.ipynb`

**Analog:** Cells 14–18 of `notebook_data.ipynb` (Phase 1 temporal split, parquet save, and EDA cells)

All Phase 2 cells must follow the same conventions already established in the notebook:

---

#### Pattern: Imports block (Cell 02, lines 1–14)

```python
import os

os.environ['MPLBACKEND'] = 'agg'
os.environ['MPLCONFIGDIR'] = '/tmp/matplotlib-config'

import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import pyarrow

print(f"pandas {pd.__version__}")
print(f"pyarrow {pyarrow.__version__}")
print(f"seaborn {sns.__version__}")
```

**Rule for Phase 2:** No new top-level imports are needed. `pandas` and `pyarrow` are already imported in Cell 02. Phase 2 cells must NOT re-import them. If `numpy` is needed for NaN sentinel (`float('nan')`), use the Python built-in — `numpy` is not imported in Cell 02.

---

#### Pattern: Section markdown header (Cells 01, 03, 05, 07, 09, 11, 13, 15, 17)

```markdown
## 10. Feature Engineering

Adiciona colunas de features pré-jogo derivadas de janelas rolling e acumuladas
por temporada. Todas as features usam `.shift(1)` antes de `.rolling()` para
excluir o resultado do jogo corrente.
```

**Rule for Phase 2:** Each logical section gets a `## N. Title` markdown cell immediately before the code cell. Section numbers continue from 9 (Cell 17 = "9. Análise exploratória…"), so Phase 2 sections start at 10.

---

#### Pattern: Function definition cell style (Cell 08 — `derive_result`)

```python
def derive_result(row):
    if row['vencedor'] == '-':
        return 'Draw'
    elif row['vencedor'] == row['mandante']:
        return 'HomeWin'
    else:
        return 'AwayWin'

df['result'] = df.apply(derive_result, axis=1)

assert df['result'].isnull().sum() == 0, "Valores nulos em 'result'"
assert set(df['result'].unique()) == {'HomeWin', 'Draw', 'AwayWin'}, \
    f"Valores inesperados: {set(df['result'].unique())}"

print(df['result'].value_counts())
```

**Rule for Phase 2:** `build_features(df)` must be defined in one cell; the call-site (apply + split + save) is a separate cell. Function docstring must be in English (matches `derive_result`'s inline comments style). Assertions are placed immediately after the operation that produces the value being asserted — not at the end of a multi-step cell.

---

#### Pattern: Parquet save + round-trip assertion (Cell 16)

```python
os.makedirs('dados', exist_ok=True)

train_df.to_parquet('dados/matches_train.parquet', engine='pyarrow', index=False)
test_df.to_parquet('dados/matches_test.parquet',   engine='pyarrow', index=False)

assert os.path.exists('dados/matches_train.parquet'), "matches_train.parquet não encontrado!"
assert os.path.exists('dados/matches_test.parquet'),  "matches_test.parquet não encontrado!"

expected_cols = ['id', 'date', 'season', 'round', 'home_team', 'away_team',
                 'home_score', 'away_score', 'home_state', 'away_state', 'result']

reload_train = pd.read_parquet('dados/matches_train.parquet')
assert len(reload_train) == 8025, f"Round-trip train: esperado 8025, obtido {len(reload_train)}"
assert list(reload_train.columns) == expected_cols, \
    f"Colunas inesperadas no train parquet: {list(reload_train.columns)}"

reload_test = pd.read_parquet('dados/matches_test.parquet')
assert len(reload_test) == 1140, f"Round-trip test: esperado 1140, obtido {len(reload_test)}"
assert list(reload_test.columns) == expected_cols, \
    f"Colunas inesperadas no test parquet: {list(reload_test.columns)}"

print(f"Salvo: treino={len(train_df)} linhas, teste={len(test_df)} linhas")
```

**Phase 2 adaptation:** Copy this pattern exactly for `feature_matrix_train.parquet` / `feature_matrix_test.parquet`. Change `expected_cols` to the 31-column list, expected row counts to (8025, 31) and (1140, 31). Add `assert fm[feature_cols].isna().sum().sum() == 0` after column list assertion.

---

#### Pattern: Temporal split (Cell 14)

```python
train_df = df[df['date'].dt.year <= 2022].copy()
test_df  = df[df['date'].dt.year >= 2023].copy()

assert len(train_df) == 8025, f"Esperado 8025 linhas de treino, obtido {len(train_df)}"
assert len(test_df)  == 1140, f"Esperado 1140 linhas de teste, obtido {len(test_df)}"
assert len(train_df) + len(test_df) == 9165, "Soma treino+teste difere do total"
assert len(set(train_df.index) & set(test_df.index)) == 0, \
    "Sobreposição de índices entre treino e teste!"
assert train_df['date'].max() < test_df['date'].min(), \
    "Sobreposição temporal entre treino e teste!"

print(f"Treino: {len(train_df)} linhas ({train_df['date'].dt.year.min()}–{train_df['date'].dt.year.max()})")
print(f"Teste:  {len(test_df)} linhas ({test_df['date'].dt.year.min()}–{test_df['date'].dt.year.max()})")
```

**Phase 2 adaptation:** Re-split the feature matrix using `feature_matrix['season'] <= 2022` (not `date.dt.year`) since `feature_matrix` is derived from the combined DataFrame and the `season` column is the canonical split key. Use the same assertion style with Portuguese-language messages.

---

#### Pattern: Assertion style throughout the notebook

Assertions follow three consistent forms:

```python
# Form 1: count check with f-string message
assert len(df) == 9165, f"Esperado 9165 linhas, obtido {len(df)}"

# Form 2: set membership check
assert set(df['result'].unique()) == {'HomeWin', 'Draw', 'AwayWin'}, \
    f"Valores inesperados: {set(df['result'].unique())}"

# Form 3: zero-null check
assert df.isnull().sum().sum() == 0, \
    f"Valores nulos encontrados após seleção: {df.isnull().sum()}"
```

**Rule for Phase 2:** Use exactly these three forms. Messages are in Portuguese when they reference row counts, column names, or output files (matching existing cells). Assertion messages that describe technical failure modes (like `"Leakage detected: ..."`) may stay in English per RESEARCH.md pattern. Never use `assert` without a message.

---

#### Pattern: Column naming convention (Cell 10 + RESEARCH.md schema)

```python
# Phase 1 columns: snake_case, Portuguese-flavored where data-source-driven
keep_cols = ['id', 'date', 'season', 'round', 'home_team', 'away_team',
             'home_score', 'away_score', 'home_state', 'away_state', 'result']
```

**Phase 2 column naming rule:** Feature columns follow `{home|away}_{stat}_{window}` convention:
- Rolling last-5: `home_goals_scored_last5`, `away_wins_last5`, `home_goal_diff_last5`
- Season cumulative: `home_win_pct_season`, `away_draw_pct_season`
- All snake_case, all lowercase. No abbreviations except established ones (`pct`, `diff`).

---

#### Pattern: Print summary at end of each code cell (all code cells)

Every code cell ends with a `print(...)` call that summarises what was produced:

```python
# Cell 04
print(f"shape: {df.shape}")
print(df.dtypes)

# Cell 14
print(f"Treino: {len(train_df)} linhas (...)")
print(f"Teste:  {len(test_df)} linhas (...)")

# Cell 16
print(f"Salvo: treino={len(train_df)} linhas, teste={len(test_df)} linhas")
```

**Rule for Phase 2:** Each code cell must end with a meaningful `print`. The feature engineering cell should print `shape`, NaN counts, and the 31 column names. The save cell must print row counts. The leakage assertion cell must print `"Leakage check PASSED: N/N ..."`.

---

#### Pattern: Headless matplotlib rendering (Cell 18)

```python
from io import BytesIO
from IPython.display import Image, display
from matplotlib.backends.backend_agg import FigureCanvasAgg
from matplotlib.figure import Figure

fig = Figure(figsize=(6, 4))
FigureCanvasAgg(fig)
ax = fig.subplots()
# ... draw ...
buffer = BytesIO()
fig.savefig(buffer, format='png', bbox_inches='tight', dpi=120)
display(Image(data=buffer.getvalue()))
```

**Rule for Phase 2 stats cell (Cell 24):** If the EDA/feature stats cell produces a plot (e.g., feature distributions by result class), use this exact headless pattern — `Figure` + `FigureCanvasAgg` + `BytesIO` + `display(Image(...))`. Do NOT use `plt.show()` or `plt.savefig()`.

---

### `dados/feature_matrix_train.parquet` (data output, batch)

**Analog:** `dados/matches_train.parquet`

Same save/load/assert pattern as Cell 16. The only differences are:
- Path: `'dados/feature_matrix_train.parquet'`
- Expected shape: `(8025, 31)` not `(8025, 11)`
- Expected columns: 11 base cols + 20 feature cols (see RESEARCH.md Output Schema)
- Additional NaN assertion on feature columns only

---

### `dados/feature_matrix_test.parquet` (data output, batch)

**Analog:** `dados/matches_test.parquet`

Same pattern as above. Expected shape: `(1140, 31)`.

---

## Shared Patterns

### Temporal split key
**Source:** `notebook_data.ipynb` Cell 14
**Apply to:** Feature matrix re-split cell (Cell 21)
```python
# Phase 1 uses date.dt.year; Phase 2 uses season column (equivalent, but season
# is the canonical groupby key already used for rolling windows)
fm_train = feature_matrix[feature_matrix['season'] <= 2022].copy()
fm_test  = feature_matrix[feature_matrix['season'] >= 2023].copy()
```

### Parquet round-trip assertion
**Source:** `notebook_data.ipynb` Cell 16
**Apply to:** Feature matrix save cell (Cell 21)
```python
reload = pd.read_parquet('dados/feature_matrix_train.parquet')
assert reload.shape == (8025, 31), f"Esperado (8025, 31), obtido {reload.shape}"
assert list(reload.columns) == expected_cols, f"Colunas inesperadas: {list(reload.columns)}"
assert reload[feature_cols].isna().sum().sum() == 0, "NaN no feature_matrix_train"
```

### Zero-null guard
**Source:** `notebook_data.ipynb` Cell 10
**Apply to:** Any cell that produces a DataFrame that must have no nulls
```python
assert df.isnull().sum().sum() == 0, \
    f"Valores nulos encontrados após seleção: {df.isnull().sum()}"
```

### `include_groups=False` in groupby apply
**Source:** RESEARCH.md Pattern 3 (verified on pandas 2.3.3)
**Apply to:** `build_features` imputation loop; leakage verification cell
```python
long.groupby(['team','season'])[src_col]
    .apply(lambda x: x.tail(10).mean(), include_groups=False)
    .reset_index()
```
Required in pandas 2.x — omitting it raises FutureWarning and errors in pandas 3.x.

### `.shift(1).rolling(5, min_periods=5)` anti-leakage pattern
**Source:** CLAUDE.md critical gotchas + RESEARCH.md Pattern 1 (verified on actual dataset)
**Apply to:** ALL rolling feature computations in `build_features`
```python
long.groupby(['team','season'])[col]
    .transform(lambda x: x.shift(1).rolling(5, min_periods=5).mean())
```
`min_periods=5` is mandatory — using `min_periods=1` would hide leakage by filling early-season rows with valid-looking floats instead of NaN.

---

## Cell Insertion Point

**Last existing cell:** Cell 18 (index 18) — EDA bar chart + baseline accuracy assertion
**Phase 2 cells start at:** Cell 19 (index 19)
**Cell count after Phase 2:** 25 cells (indices 0–24)

Planned cell sequence (from RESEARCH.md Cell Structure):

| New Index | Type | Section Number | Purpose |
|-----------|------|----------------|---------|
| 19 | markdown | `## 10. Feature Engineering` | Section header |
| 20 | code | — | `build_features(df)` definition |
| 21 | code | — | Concat → build_features → re-split → save + round-trip assertions |
| 22 | markdown | `## 11. Verificação de Vazamento` | Section header |
| 23 | code | — | Leakage assertion (raw pre-imputation round-1 NaN check) |
| 24 | code | — | Feature stats: `.describe()` and class-conditional means |

---

## No Analog Found

No files are without analog. All patterns are derivable from existing Phase 1 cells.

---

## Metadata

**Analog search scope:** `notebook_data.ipynb` (all 19 cells read in full)
**Files scanned:** 1 notebook + 2 context files
**Pattern extraction date:** 2026-05-22
