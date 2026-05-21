# Coding Conventions

**Analysis Date:** 2026-05-21

## Project Type

This is a single-notebook ML research project. The primary deliverable is `notebook.ipynb`. There are no standalone Python source modules. All conventions apply to Jupyter notebook cells.

## Naming Patterns

**Variables:**
- `snake_case` throughout: `x_train`, `y_train`, `x_extra`, `x_train_final`, `loss_treino`, `loss_validacao`
- Training/data arrays follow the pattern `x_<descriptor>` and `y_<descriptor>`
- Phase-specific variables use suffix: `_fase1`, `_fase2` (e.g., `x_extra_rep_fase1`, `history_fase1`, `history_fase2`)
- Inverted datasets use `_inv` suffix: `x_train_inv`, `x_test_inv`

**History objects:**
- Named `history_<phase>`: `history_fase1`, `history_fase2`

**Callbacks:**
- `early_stopping` (snake_case)

**Layers/Model:**
- Model variable named `model`
- No custom layer subclasses — only Keras built-ins via `Sequential`

**Plot variables:**
- `loss_treino`, `loss_validacao` (Portuguese-language labels)
- `epocas`, `fim_fase1`

## Language

Mixed Portuguese/English. Variable names and inline comments trend toward Portuguese for domain labels (e.g., `loss_treino`, `loss_validacao`, `epocas`, labels on plots). Standard Python/Keras API identifiers remain in English.

## Code Style

**Formatting:**
- No automated formatter configured (no `.prettierrc`, `black`, `ruff`, or `isort` config detected)
- Standard PEP 8 spacing applied manually

**Linting:**
- No linter configured (no `.flake8`, `.pylintrc`, `mypy.ini`, or `pyproject.toml [tool.*]` lint sections)

**Line length:**
- No enforced limit; cells use natural notebook line widths

## Import Organization

**Order (as observed in `notebook.ipynb`):**
1. Standard library: `os`, `zipfile`, `io.BytesIO`
2. Third-party data/ML: `numpy`, `tensorflow`, `keras`, `matplotlib`
3. Keras sub-modules imported individually: `Sequential`, `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`, `EarlyStopping`, `Adam`

**Style:**
- Mix of top-level (`import numpy as np`) and sub-module (`from tensorflow.keras.layers import Dense`) imports
- Keras objects are always imported from `tensorflow.keras` (not standalone `keras` package directly)
- `matplotlib` uses non-interactive backend via `os.environ['MPLBACKEND'] = 'agg'` set before import in the plotting cell

## Error Handling

No explicit `try/except` blocks in the notebook. Error handling is implicit — Keras/TensorFlow raises on invalid input or constraint violations. No custom validation of model outputs or data shapes beyond `print()` diagnostics.

## Logging / Diagnostics

**Pattern:** `print()` with f-strings for shape/dtype diagnostics:
```python
print(f"train: {x_train.shape}, dtype={x_train.dtype}, range=[{x_train.min()}, {x_train.max()}]")
print(f"Acurácia no conjunto de treino: {train_accuracy * 100:.2f}%")
```

No structured logging framework is used.

## Comments

**Inline comments:** Used on key hyperparameter lines to explain the choice:
```python
optimizer=Adam(learning_rate=0.0005),  # comment
patience=10,           # comment
```

**Markdown cells:** Each major section is introduced with a `##`-level markdown heading explaining the intent. Documentation quality is a graded criterion.

**TODO markers:** Used to mark stub cells that require implementation (`# TODO: train your model`, `# TODO: evaluate`).

## Notebook Cell Organization

Cells follow this fixed sequence as defined by the grading contract:
1. Markdown title/author block
2. Markdown section heading
3. Code cell (data loading)
4. Markdown section heading
5. Code cell (model definition)
6. Markdown section heading
7. Code cell (training — two-phase)
8. Code cell (loss curve plot)
9. Markdown section heading
10. Code cell (evaluation)

## Model Definition Pattern

Models are defined using `keras.Sequential` with layers listed inline. No functional API, no custom `tf.Module` subclasses. All preprocessing (e.g., `Rescaling`) is baked in as the first layer so the model accepts raw `uint8` inputs.

```python
model = Sequential([
    Rescaling(1./255, input_shape=(28, 28)),
    Flatten(),
    Dense(N, activation='relu'),
    BatchNormalization(),
    Dropout(rate),
    ...
    Dense(10, activation='softmax'),
])
```

## Hyperparameter Style

Hyperparameters are written as keyword arguments with inline comments explaining the value:
```python
optimizer=Adam(learning_rate=0.0005),  # learning rate chosen for phase 1
patience=10,  # stops early if no improvement
```

## Allowed Layers (Contract Constraint)

Only these Keras layers may be used (graded requirement):
- `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`

---

*Convention analysis: 2026-05-21*
