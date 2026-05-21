# Phase 1: Data Ingestion & Cleaning - Pattern Map

**Mapped:** 2026-05-21
**Files analyzed:** 3 (1 notebook to create, 2 parquet outputs)
**Analogs found:** 1 / 3 (notebook analog only; parquet outputs are pure data artifacts with no code analog)

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `notebook_data.ipynb` | notebook (data pipeline) | batch / transform | `notebook.ipynb` | role-match (same notebook format; different domain) |
| `dados/matches_train.parquet` | data artifact | batch | — | no analog (first parquet in repo) |
| `dados/matches_test.parquet` | data artifact | batch | — | no analog (first parquet in repo) |

---

## Pattern Assignments

### `notebook_data.ipynb` (notebook, batch/transform)

**Analog:** `notebook.ipynb`

**Imports / env setup pattern** (notebook.ipynb, cell `0967d16e`, lines 1-5):

```python
import os
from io import BytesIO

os.environ['MPLBACKEND'] = 'agg'
os.environ['MPLCONFIGDIR'] = '/tmp/matplotlib-config'
```

The env vars MUST be set before any matplotlib import. This is the established convention in this repo for headless rendering in Jupyter. Apply to `notebook_data.ipynb` Cell 1 (Setup), placing `os.environ` assignments before `import matplotlib.pyplot as plt` and `import seaborn as sns`.

**Core data-load pattern** (notebook.ipynb, cell `8b3f3fbb`):

```python
import numpy as np
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

(x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()
print(f"train: {x_train.shape}, dtype={x_train.dtype}, range=[{x_train.min()}, {x_train.max()}]")
print(f"test : {x_test.shape}, dtype={x_test.dtype}")
```

The pattern to copy: load data, then immediately print shape/dtype to produce a visible cell output that acts as a smoke-test. Adapt for pandas:

```python
df = pd.read_csv('dados/archive/campeonato-brasileiro-full.csv')
print(f"shape: {df.shape}, dtypes:\n{df.dtypes}")
assert len(df) == 9165, f"Expected 9165 rows, got {len(df)}"
```

**Portuguese-label convention** (notebook.ipynb, cell `0967d16e`):

```python
loss_treino = history_fase1.history['loss'] + history_fase2.history['loss']
loss_validacao = history_fase1.history['val_loss'] + history_fase2.history['val_loss']
# ...
ax.plot(epocas, loss_treino, marker='o', label='Loss treino')
ax.plot(epocas, loss_validacao, marker='o', label='Loss validação')
ax.set_title('Loss em todas as épocas de treinamento')
ax.set_xlabel('Época')
ax.set_ylabel('Loss')
```

Convention: axis labels and legend text use Portuguese. Column names in computed DataFrames use English (`home_team`, `result`). Apply to EDA cell: `ax.set_title('Distribuição dos resultados (2003–2025)')`, `ax.set_xlabel('Resultado')`, `ax.set_ylabel('Contagem')`.

**Matplotlib headless render pattern** (notebook.ipynb, cell `0967d16e`):

```python
from IPython.display import Image, display
from matplotlib.backends.backend_agg import FigureCanvasAgg
from matplotlib.figure import Figure

fig = Figure(figsize=(10, 5))
FigureCanvasAgg(fig)
ax = fig.subplots()
# ... plot calls ...
buffer = BytesIO()
fig.savefig(buffer, format='png', bbox_inches='tight', dpi=120)
display(Image(data=buffer.getvalue()))
```

This is the established repo pattern for headless rendering. Use it for the EDA bar chart cell if seaborn's `plt.show()` fails in the CI environment. Alternatively, `sns.countplot` with `plt.savefig` + `display(Image(...))` follows the same approach.

**Evaluation print pattern** (notebook.ipynb, cell `3fd76014`):

```python
_, train_accuracy = model.evaluate(x_train_final, y_train_final, verbose=0)
test_loss, test_accuracy = model.evaluate(x_test_inv, y_test, verbose=0)
print(f"Acurácia no conjunto de treino: {train_accuracy * 100:.2f}%")
print(f"Acurácia no conjunto de teste: {test_accuracy * 100:.2f}%")
```

Convention: final summary cells use `print(f"...")` with percentage formatting. Adapt for the parquet-save confirmation cell:

```python
print(f"Salvo: treino={len(train_df)} linhas, teste={len(test_df)} linhas")
print(f"Distribuição (treino): {train_df['result'].value_counts().to_dict()}")
```

---

## Shared Patterns

### Matplotlib Headless Backend
**Source:** `notebook.ipynb`, cell `0967d16e`, lines 1-4
**Apply to:** All cells in `notebook_data.ipynb` that import matplotlib or seaborn

```python
import os
os.environ['MPLBACKEND'] = 'agg'
os.environ['MPLCONFIGDIR'] = '/tmp/matplotlib-config'
```

Set these two env vars at the top of Cell 1 (Setup) before any plot library import. This is a hard repo requirement — without it, matplotlib fails in headless CI environments.

### Print-Based Smoke Tests After Each Major Step
**Source:** `notebook.ipynb`, cells `8b3f3fbb` and `3fd76014`
**Apply to:** Every cell in `notebook_data.ipynb` that transforms data

The repo convention is: after every load or transform, print the resulting shape and a key invariant. Pattern:

```python
# After load:
print(f"shape: {df.shape}")
# After split:
print(f"train={len(train_df)}, test={len(test_df)}")
# After save:
print(f"Salvo: treino={len(train_df)} linhas, teste={len(test_df)} linhas")
```

These print statements are the "evaluation" step for a data pipeline — they make cell output a human-readable audit trail.

### Portuguese Labels for Display Output
**Source:** `notebook.ipynb`, cell `0967d16e`
**Apply to:** All `ax.set_title`, `ax.set_xlabel`, `ax.set_ylabel`, `ax.legend`, and `print(f"...")` summary strings in `notebook_data.ipynb`

English for column names and variable names. Portuguese for chart labels, axis labels, and printed summaries. Examples:
- Variable name: `train_df`, `test_df`, `canonical_names` (English)
- Chart title: `'Distribuição dos resultados (2003–2025)'` (Portuguese)
- Print: `f"Acurácia da divisão temporal: treino até 2022"` (Portuguese)

### Assert-Then-Print Inline Testing
**Source:** `notebook.ipynb`, cell `8b3f3fbb` (implicit: shape checks)
**Apply to:** Every data transformation cell in `notebook_data.ipynb`

The notebook has no separate test file. All verification is inline `assert` statements in notebook cells. The execution of the whole notebook by `pytest --nbmake` treats any unhandled exception (including `AssertionError`) as a test failure. Pattern:

```python
# Load cell:
assert len(df) == 9165, f"Expected 9165 rows, got {len(df)}"

# Split cell:
assert len(train_df) == 8025, f"Expected 8025 train rows, got {len(train_df)}"
assert len(test_df)  == 1140, f"Expected 1140 test rows, got {len(test_df)}"
assert len(set(train_df.index) & set(test_df.index)) == 0, "Train/test index overlap!"

# Save cell:
assert os.path.exists('dados/matches_train.parquet')
assert os.path.exists('dados/matches_test.parquet')
```

---

## No Analog Found

Files where no close codebase match exists (use RESEARCH.md patterns instead):

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| `dados/matches_train.parquet` | data artifact | batch | No parquet files exist in this repo yet; pattern comes from `pandas.DataFrame.to_parquet` API |
| `dados/matches_test.parquet` | data artifact | batch | Same as above |

For both parquet outputs, the complete pattern is in RESEARCH.md Pattern 4 (lines 248-257):

```python
import os
os.makedirs('dados', exist_ok=True)
train_df.to_parquet('dados/matches_train.parquet', engine='pyarrow', index=False)
test_df.to_parquet('dados/matches_test.parquet',   engine='pyarrow', index=False)
assert os.path.exists('dados/matches_train.parquet')
assert os.path.exists('dados/matches_test.parquet')
print(f"Salvo: treino={len(train_df)} linhas, teste={len(test_df)} linhas")
```

---

## Key Anti-Patterns to Avoid (from RESEARCH.md)

These anti-patterns have no codebase analog to avoid — they are domain-specific to this notebook:

| Anti-Pattern | Why Wrong | Correct Pattern |
|--------------|-----------|-----------------|
| `read_csv(parse_dates=['data'])` | Deprecated `infer_datetime_format` path in pandas 2.x | `pd.to_datetime(df['data'], format='%d/%m/%Y')` after load |
| `df.sort_values('data')` (string column) | DD/MM/YYYY sorts lexicographically wrong | Parse date first, then `assert df['date'].is_monotonic_increasing` |
| `df.dropna()` on full DataFrame | Drops 96% of rows (formacao/arrecadacao columns have thousands of nulls) | Drop unused columns by name first; the core columns have 0 nulls |
| `shuffle=True` or `KFold` for split | Leaks future data into training set | Year-cutoff only: `df[df['date'].dt.year <= 2022]` |
| Asserting `len(train_df) == 7200` | Actual is 8,025 (COVID years cause anomalies) | Assert exact verified counts: 8025 / 1140 |

---

## Metadata

**Analog search scope:** `/home/theo/AI/Projeto_2_AI/` (project root only — no subdirectories contain Python source)
**Files scanned:** 1 (`notebook.ipynb` — the only code file in the repo)
**Pattern extraction date:** 2026-05-21
