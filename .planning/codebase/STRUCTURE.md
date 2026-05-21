# Structure

*Last mapped: 2026-05-21*

## Directory Layout

```
/home/theo/AI/Projeto_2_AI/
├── notebook.ipynb          # Primary deliverable — all ML code lives here
├── pyproject.toml          # Python project config, dependencies (UV-managed)
├── uv.lock                 # UV lockfile for reproducible installs
├── CLAUDE.md               # Project instructions for Claude Code
├── README.md               # Minimal readme
├── dados/                  # Legacy data directory (Brazilian football CSVs)
│   └── archive/            # CSV files — NOT used by current notebook
└── .venv/                  # UV-managed virtual environment
```

## Key Files

| File | Purpose |
|------|---------|
| `notebook.ipynb` | Entire ML pipeline: data loading → model definition → training → evaluation |
| `pyproject.toml` | Declares Python version constraint (>=3.11, <3.13) and dependencies |
| `uv.lock` | Lockfile ensuring reproducible installs |
| `CLAUDE.md` | Constraints and architecture guidance for AI-assisted development |

## Notebook Cell Organization

The notebook is the single source of truth for all code:

1. **Title / Context cell** — Project description and grading constraints
2. **Data loading** — MNIST from `keras.datasets`, external validation set via HTTP
3. **Model definition** — Sequential Keras model with allowed layers only
4. **Phase 1 training** — Train on inverted MNIST + external validation set
5. **Phase 2 fine-tuning** — Fine-tune exclusively on external validation set
6. **Evaluation metrics** — Accuracy scores, predictions
7. **Loss curve plots** — Visualization of training/validation progress

## Naming Conventions

- Single notebook file — no module structure
- Variables: `snake_case` (Python convention)
- Model layers follow Keras naming (e.g., `Dense`, `Flatten`, `BatchNormalization`)
- External data: `x_val`, `y_val` (validation set loaded via HTTP)

## Where to Add New Code

- All ML experiments go in `notebook.ipynb`
- New dependencies declared in `pyproject.toml` under `[project] dependencies`
- No separate `src/` or module structure — this is a notebook-first project

## What NOT to Add

- New Python modules or packages (notebook-only project)
- `dados/` directory files (legacy, unused)
- Test files (no test framework; verification is external)

---
*Mapped: 2026-05-21*
