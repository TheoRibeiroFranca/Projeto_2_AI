# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projects in This Repo

**Active: Brazilian League Match Predictor** (GSD-managed — see `.planning/`)
Predicts W/D/L outcomes for Brazilian Série A matches using rolling-window form features and scikit-learn classifiers. Three separate model notebooks: Logistic Regression, Random Forest, Gradient Boosting. Target: ~65-70% accuracy.

**Legacy: MNIST Digit Classifier** (`notebook.ipynb`)
Insper AI trainee deliverable. Two-phase Keras training on inverted MNIST. Input `(28,28)` uint8 → `(10,)` softmax. Weight file < 800 KB. Allowed layers: `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`.

## Setup & Running

```bash
# Install base dependencies
uv sync
uv add scikit-learn pyarrow seaborn pytest nbmake  # football predictor deps (not yet in pyproject.toml)

# Activate virtual environment
source .venv/bin/activate

# Run Phase 1 data pipeline (produces parquet files)
jupyter notebook notebook_data.ipynb

# Test notebook execution end-to-end
pytest --nbmake notebook_data.ipynb -x

# Launch football predictor model notebooks (created during execution phases)
jupyter notebook notebook_logistic.ipynb        # Logistic Regression baseline
jupyter notebook notebook_random_forest.ipynb   # Random Forest
jupyter notebook notebook_gradient_boost.ipynb  # Gradient Boosting (target: 65-70%)

# Launch legacy MNIST notebook
jupyter notebook notebook.ipynb
```

Dependencies are managed with **UV** (`pyproject.toml` + `uv.lock`). Python >=3.11, <3.13 is required.

## Football Predictor — Critical Gotchas

**Temporal leakage (top risk — inflates accuracy 10-20 pp silently):**
- Always use `.shift(1)` before `.rolling()` — never include the current match result in windows
- Train/test split must use a fixed date cutoff, never `shuffle=True` or `KFold`
- Do NOT join in-game stats (shots, possession) without confirming they are pre-match values

**Draw class collapse:**
- Always set `class_weight='balanced'` before the first `fit()` call
- Always output `classification_report` — overall accuracy alone hides zero Draw recall

**Team name normalization:**
- Build a canonical name dict before any `groupby` — 20+ seasons produce many variants (e.g. "Atletico-MG" vs "Atletico MG")
- Assert top clubs have 380+ match rows after normalization

**Evaluation:**
- Use `TimeSeriesSplit` exclusively — never `KFold` or `StratifiedKFold`
- Compare against naive "always Home Win" baseline (49.6% — verified against actual dataset; the ~46% figure is wrong) before claiming success
- Report macro-F1 alongside accuracy

## Data

`dados/archive/` contains Brazilian football championship CSVs used by the football predictor:
- `campeonato-brasileiro-full.csv` — primary source (9,165 matches, 2003–2025)
- `campeonato-brasileiro-estatisticas-full.csv` — shot/possession stats (zero-filled pre-2013, use with caution)
- `campeonato-brasileiro-gols.csv` — goal-level event data
- `campeonato-brasileiro-cartoes.csv` — card-level event data

The legacy MNIST notebook loads data from `keras.datasets` and a fixed HTTP URL.

## Football Predictor — Output Contract

`notebook_data.ipynb` (Phase 1) produces two parquet files consumed by all model notebooks:

| File | Rows | Period |
|------|------|--------|
| `dados/matches_train.parquet` | 8,025 | 2003–2022 |
| `dados/matches_test.parquet` | 1,140 | 2023–2025 |

**Schema (11 columns):** `id`, `date`, `season`, `round`, `home_team`, `away_team`, `home_score`, `away_score`, `home_state`, `away_state`, `result`

**`result` encoding** (must be consistent across all 3 model notebooks):
- `'HomeWin'` — home team won
- `'Draw'` — match ended level
- `'AwayWin'` — away team won

**Do not re-derive `result` or re-run cleaning in model notebooks — load from parquet.**

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

- **Python** >=3.11, <3.13 — runtime resolves to 3.12.x; managed via UV (`pyproject.toml` + `uv.lock`)
- **Jupyter Notebook** — all deliverables are notebooks; no importable `.py` modules
- **Football predictor deps:** `pandas>=2.1`, `scikit-learn`, `pyarrow` (parquet), `seaborn` (EDA), `pytest` + `nbmake` (notebook testing)
- **Legacy MNIST deps:** `tensorflow==2.18.*`, `keras==3.6.*`, `numpy>=1.26,<2.2`, `matplotlib>=3.8`
- **No GPU:** TensorFlow runs CPU-only; no CUDA drivers
- **No `.env`:** no environment variables required; notebooks set `MPLBACKEND=agg` / `MPLCONFIGDIR=/tmp/matplotlib-config` at runtime for headless rendering
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

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
## Code Style
- No automated formatter configured (no `.prettierrc`, `black`, `ruff`, or `isort` config detected)
- Standard PEP 8 spacing applied manually
- No linter configured (no `.flake8`, `.pylintrc`, `mypy.ini`, or `pyproject.toml [tool.*]` lint sections)
- No enforced limit; cells use natural notebook line widths
## Import Organization
- Mix of top-level (`import numpy as np`) and sub-module (`from tensorflow.keras.layers import Dense`) imports
- Keras objects are always imported from `tensorflow.keras` (not standalone `keras` package directly)
- `matplotlib` uses non-interactive backend via `os.environ['MPLBACKEND'] = 'agg'` set before import in the plotting cell
## Allowed Layers (Contract Constraint — MNIST only)
- `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

> **Note:** This section describes the legacy MNIST notebook (`notebook.ipynb`). Football predictor architecture lives in `.planning/` (ROADMAP.md, phases/).

## System Overview
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
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
