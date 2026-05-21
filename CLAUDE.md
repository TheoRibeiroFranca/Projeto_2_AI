# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Insper AI trainee final project for MNIST digit classification using a neural network with strict architectural constraints. The primary deliverable is `notebook.ipynb`.

## Setup & Running

```bash
# Install dependencies
uv sync

# Activate virtual environment
source .venv/bin/activate

# Launch the notebook
jupyter notebook notebook.ipynb
```

Dependencies are managed with **UV** (`pyproject.toml` + `uv.lock`). Python >=3.11, <3.13 is required.

## Model Constraints (Graded Requirements)

The model submitted for evaluation must satisfy:
- **Input:** `(28, 28)` uint8 images (pixel values 0–255)
- **Output:** `(10,)` softmax probabilities (one per digit class)
- **Weight file size:** < 800 KB
- **Allowed layers only:** `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`

## Architecture & Training Strategy

The notebook follows a two-phase training approach:

**Phase 1** — Train on inverted MNIST images plus an external validation set:
- Adam optimizer (lr=0.0005), batch size 256, up to 10 epochs
- EarlyStopping with patience=10

**Phase 2** — Fine-tune exclusively on the external validation set (replicated):
- Adam optimizer (lr=0.00005), batch size 16, up to 40 epochs
- EarlyStopping with patience=15

The external validation set (`x_val`, `y_val`) is loaded via HTTP from a fixed URL at the top of the training cells. Data augmentation via image inversion is applied during Phase 1 to improve generalization.

## Data

`dados/archive/` contains Brazilian football championship CSVs — these are **not used** by the current notebook (legacy from an earlier project version). The actual training data is MNIST loaded directly from `keras.datasets`.

## Notebook Cell Organization

The notebook is structured as: title/context → data loading → model definition → two-phase training → evaluation metrics → loss curve plots. Grading considers both documentation quality within cells and the leaderboard accuracy score.

<!-- GSD:project-start source:PROJECT.md -->
## Project

**Brazilian League Match Predictor**

A Jupyter notebook-based ML system that predicts the outcome (Home Win / Draw / Away Win) of Brazilian football championship matches. Given two teams and their recent form — goals scored/conceded over their last 5 matches, win/draw/loss streak, and home vs. away records — the model classifies the most likely result. Data comes from the CSV files already in the repo supplemented by a live external API.

**Core Value:** Given any two Brazilian league teams, predict the match outcome with meaningful accuracy — outperforming naive baselines and approaching the ~65-70% range typical of sports prediction models.

### Constraints

- **Delivery format:** Jupyter notebook (consistent with trainee program format)
- **Data:** Must work with existing CSVs as baseline; API supplements for recency
- **Target accuracy:** ~65-70% on held-out test — competitive with sports prediction baselines
- **No hard grading constraints:** Free to choose architecture, no layer/size restrictions
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Languages
- Python >=3.11, <3.13 - All project code; runtime resolves to 3.12.x in the active venv
## Runtime
- CPython 3.12.13 (system Python; venv at `.venv/`)
- UV — managed via `pyproject.toml` + `uv.lock`
- Lockfile: present (`uv.lock`)
## Frameworks
- TensorFlow 2.18.1 — backend compute engine; CPU-only (no CUDA drivers detected)
- Keras 3.6.0 — high-level model API; used directly via `tensorflow.keras`
- `keras.datasets.mnist` — built-in MNIST dataset loader
- `tf.keras.utils.image_dataset_from_directory` — loads the external validation set from a local directory
- `tensorflow.keras.callbacks.EarlyStopping` — training callbacks
- `tensorflow.keras.optimizers.Adam` — optimizer
- JupyterLab / Jupyter Notebook — `ipykernel` 7.2.0, `jupyter-client` 8.8.0, `jupyter-core` 5.9.1
- IPython 9.13.0 — kernel
- matplotlib 3.8+ (with `matplotlib-inline` 0.1.7) — loss curve plots rendered via `FigureCanvasAgg` (headless, no display required)
- Pillow 12.2.0 — image support (matplotlib backend dependency)
- UV — dependency resolution and virtual environment management
- debugpy 1.8.20 — notebook debugging support
## Key Dependencies
- `tensorflow==2.18.*` — model construction, training, serialization
- `keras==3.6.*` — Sequential API, layer types, callbacks
- `numpy>=1.26,<2.2` — array operations; resolved to 2.0.2 in lockfile
- `requests>=2.32` — HTTP client; resolved to 2.33.1 (present in `pyproject.toml` but not actively used in current notebook cells)
- `h5py` 3.16.0 — HDF5 model weight serialization (Keras `.h5` saves)
- `grpcio` 1.80.0 — TensorBoard/gRPC transport
- `tensorboard` 2.18.0 — training metrics visualization (optional)
- `protobuf` 5.29.6 — TensorFlow serialization format
- `tensorflow-io-gcs-filesystem` 0.37.1 — GCS dataset access (bundled with TF)
## Configuration
- No `.env` file present
- No environment variables required for local training; TensorFlow uses `TF_ENABLE_ONEDNN_OPTS` (optional, noted in logs)
- Notebook sets `MPLBACKEND=agg` and `MPLCONFIGDIR=/tmp/matplotlib-config` at runtime for headless rendering
- `pyproject.toml` — project metadata and direct dependencies
- `uv.lock` — fully pinned transitive dependency tree
- `[tool.uv] package = false` — project is not installed as a package; dependencies only
## Platform Requirements
- Python >=3.11, <3.13
- UV installed (`uv sync` installs all dependencies into `.venv/`)
- Jupyter-compatible environment (`jupyter notebook notebook.ipynb`)
- CPU-only execution confirmed; GPU support not available in current environment
- No server deployment; deliverable is `notebook.ipynb`
- Model weight file must be < 800 KB (grading constraint)
- Model must accept `(28, 28)` uint8 input and output `(10,)` softmax probabilities
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## Project Type
## Naming Patterns
- `snake_case` throughout: `x_train`, `y_train`, `x_extra`, `x_train_final`, `loss_treino`, `loss_validacao`
- Training/data arrays follow the pattern `x_<descriptor>` and `y_<descriptor>`
- Phase-specific variables use suffix: `_fase1`, `_fase2` (e.g., `x_extra_rep_fase1`, `history_fase1`, `history_fase2`)
- Inverted datasets use `_inv` suffix: `x_train_inv`, `x_test_inv`
- Named `history_<phase>`: `history_fase1`, `history_fase2`
- `early_stopping` (snake_case)
- Model variable named `model`
- No custom layer subclasses — only Keras built-ins via `Sequential`
- `loss_treino`, `loss_validacao` (Portuguese-language labels)
- `epocas`, `fim_fase1`
## Language
## Code Style
- No automated formatter configured (no `.prettierrc`, `black`, `ruff`, or `isort` config detected)
- Standard PEP 8 spacing applied manually
- No linter configured (no `.flake8`, `.pylintrc`, `mypy.ini`, or `pyproject.toml [tool.*]` lint sections)
- No enforced limit; cells use natural notebook line widths
## Import Organization
- Mix of top-level (`import numpy as np`) and sub-module (`from tensorflow.keras.layers import Dense`) imports
- Keras objects are always imported from `tensorflow.keras` (not standalone `keras` package directly)
- `matplotlib` uses non-interactive backend via `os.environ['MPLBACKEND'] = 'agg'` set before import in the plotting cell
## Error Handling
## Logging / Diagnostics
## Comments
## Notebook Cell Organization
## Model Definition Pattern
## Hyperparameter Style
## Allowed Layers (Contract Constraint)
- `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## System Overview
```text
```
## Component Responsibilities
| Component | Responsibility | File |
|-----------|----------------|------|
| Data Loading | Load MNIST via keras, extract external validation zip | `notebook.ipynb` cell `8b3f3fbb`, `35d075b9` |
| Data Augmentation | Invert pixel values (255 - x) on train and test sets | `notebook.ipynb` cell `35d075b9` |
| External Validation Loader | Read `validation-set.zip` into numpy arrays | `notebook.ipynb` cell `35d075b9` |
| Model Definition | Keras Sequential with preprocessing baked in | `notebook.ipynb` cell `7a84b337` |
| Phase 1 Training | Broad generalization on inverted MNIST + external data | `notebook.ipynb` cell `35d075b9` |
| Phase 2 Fine-tuning | Domain adaptation on external validation set only | `notebook.ipynb` cell `35d075b9` |
| Evaluation | Accuracy metrics on train and test sets | `notebook.ipynb` cell `3fd76014` |
| Visualization | Combined loss curve spanning both phases | `notebook.ipynb` cell `0967d16e` |
## Pattern Overview
- All logic lives in `notebook.ipynb`; there are no importable Python modules
- Preprocessing is baked into the model as `Rescaling` and `Flatten` layers (grading requirement)
- Two-phase training separates broad generalization (Phase 1) from domain-specific fine-tuning (Phase 2)
- Data augmentation uses pixel inversion to simulate the external validation set's domain
## Layers
- Purpose: Load, augment, and concatenate training data
- Location: `notebook.ipynb` cells `8b3f3fbb` and `35d075b9`
- Contains: MNIST loading, zip extraction, numpy array assembly, pixel inversion
- Depends on: `keras.datasets.mnist`, `zipfile`, `tensorflow.keras.utils.image_dataset_from_directory`
- Used by: Training orchestration layer
- Purpose: Declare the neural network architecture as a Keras Sequential model
- Location: `notebook.ipynb` cell `7a84b337`
- Contains: `Rescaling(1/255)`, `Flatten`, four `Dense→BatchNormalization→Dropout` blocks, softmax output
- Allowed layers only: `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`
- Depends on: `tensorflow.keras`
- Used by: Training orchestration layer
- Purpose: Execute two-phase training with separate hyperparameters and data distributions
- Location: `notebook.ipynb` cell `35d075b9`
- Contains: `model.fit` calls for Phase 1 and Phase 2, EarlyStopping callbacks, data tiling logic
- Depends on: Model Definition layer, Data Ingestion layer
- Used by: Evaluation layer
- Purpose: Report accuracy and render loss curves
- Location: `notebook.ipynb` cells `3fd76014` and `0967d16e`
- Contains: `model.evaluate`, matplotlib figure combining both phase histories
- Depends on: Trained model, history objects from both phases
- Used by: Grader (notebook output)
## Data Flow
### Primary Training Path
### Inference / Evaluation Path
- Model weights are held in-memory in the `model` variable throughout the notebook session
- Training history is stored in `history_fase1` and `history_fase2` dict-like objects
- No model serialization (saving/loading weights) is present in the current notebook
## Key Abstractions
- Purpose: Encapsulates the full inference pipeline including preprocessing
- Location: `notebook.ipynb` cell `7a84b337`
- Pattern: `model = Sequential([Rescaling(1/255), Flatten(), Dense(200, 'relu'), BatchNormalization(), Dropout(0.1), ...])`
- Purpose: Produce a single continuous loss curve across both training phases
- Location: `notebook.ipynb` cell `0967d16e`
- Pattern: Concatenate `history_fase1.history['loss'] + history_fase2.history['loss']`, draw `axvline` at `fim_fase1`
- Purpose: Domain adaptation — external validation set uses dark-on-light images; MNIST uses light-on-dark
- Location: `notebook.ipynb` cell `35d075b9`
- Pattern: `x_train_inv = 255 - x_train`
## Entry Points
- Location: `notebook.ipynb`
- Triggers: `jupyter notebook notebook.ipynb` then "Run All Cells"
- Responsibilities: Executes all data loading, training, evaluation, and visualization in cell order
## Architectural Constraints
- **Allowed layers:** Only `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout` (grading contract)
- **Input contract:** Model must accept `(28, 28)` uint8 tensors (values 0–255); preprocessing must be internal
- **Output contract:** Model must produce `(10,)` softmax probability vector
- **Weight budget:** Total parameters (trainable + non-trainable) must serialize to < 800 KB
- **No module imports:** No `.py` source files exist; all logic is inline in the notebook
- **External data dependency:** Phase 1 training references `x_sint` / `y_sint` variables that are not defined in the visible cells — this is an unresolved reference in the current notebook state
- **No GPU:** TensorFlow runs CPU-only (CUDA drivers not found in execution environment)
## Anti-Patterns
### Undefined Variable Reference (`x_sint` / `y_sint`)
### No Model Persistence
### Final Dense Layer Without Softmax Activation
## Error Handling
- No explicit error handling around zip extraction or HTTP data loading
- No input validation on external dataset shape before concatenation
## Cross-Cutting Concerns
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
