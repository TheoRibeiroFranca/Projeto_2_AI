# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projects in This Repo

**Active: Brazilian League Match Predictor** (GSD-managed — see `.planning/`)
Predicts W/D/L outcomes for Brazilian Série A matches using rolling-window form features and scikit-learn classifiers. Three separate model notebooks: Logistic Regression, Random Forest, Gradient Boosting. Target: ~65-70% accuracy.
Phase status: Phase 1 (data ingestion) ✓ | Phase 2 (feature engineering) ✓ | Phase 3 (baseline models) pending | Phase 4 (GBM + polish) pending

**Legacy: MNIST Digit Classifier** (`notebook.ipynb`)
Insper AI trainee deliverable. Two-phase Keras training on inverted MNIST. Input `(28,28)` uint8 → `(10,)` softmax. Weight file < 800 KB. Allowed layers: `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`.

## Setup & Running

```bash
# Install all dependencies (football predictor + MNIST deps all in pyproject.toml)
uv sync

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

## Football Predictor — Feature Matrix Contract

`notebook_data.ipynb` (Phase 2) produces the feature parquet files that **all model notebooks must load**:

| File | Rows | Columns |
|------|------|---------|
| `dados/feature_matrix_train.parquet` | 8,025 | 31 |
| `dados/feature_matrix_test.parquet` | 1,140 | 31 |

**Schema:** 11 base columns (same as matches parquet) plus 20 feature columns:

| Feature | Description |
|---------|-------------|
| `home_goals_scored_last5` / `away_goals_scored_last5` | Rolling 5-match goals scored |
| `home_goals_conceded_last5` / `away_goals_conceded_last5` | Rolling 5-match goals conceded |
| `home_wins_last5` / `home_draws_last5` / `home_losses_last5` | Home form (last 5) |
| `away_wins_last5` / `away_draws_last5` / `away_losses_last5` | Away form (last 5) |
| `home_win_pct_season` / `home_draw_pct_season` / `home_loss_pct_season` | Season win/draw/loss % (home) |
| `away_win_pct_season` / `away_draw_pct_season` / `away_loss_pct_season` | Season win/draw/loss % (away) |
| `home_goal_diff_last5` / `away_goal_diff_last5` | Goal difference last 5 |
| `home_points_last5` / `away_points_last5` | Points accumulated last 5 |

**Model notebooks must load `feature_matrix_*.parquet`, not `matches_*.parquet`.** Round-1 rows have NaN features — drop or impute before fitting.

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

**Football predictor:** Architecture and phase plans live in `.planning/` (ROADMAP.md, phases/). Data flow: `campeonato-brasileiro-full.csv` → `notebook_data.ipynb` → `feature_matrix_{train,test}.parquet` → model notebooks.

**Legacy MNIST (`notebook.ipynb`):** All logic is inline (no `.py` modules). Two-phase training: Phase 1 broad generalization on inverted MNIST + external data; Phase 2 fine-tuning on external validation set only. Preprocessing (`Rescaling(1/255)`, `Flatten`) is baked into the model. Key constraints: allowed layers only (`Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`), input `(28,28)` uint8, output `(10,)` softmax, weight file < 800 KB. Training history in `history_fase1` / `history_fase2`; no model serialization.
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
