# Technology Stack

**Analysis Date:** 2026-05-21

## Languages

**Primary:**
- Python >=3.11, <3.13 - All project code; runtime resolves to 3.12.x in the active venv

## Runtime

**Environment:**
- CPython 3.12.13 (system Python; venv at `.venv/`)

**Package Manager:**
- UV — managed via `pyproject.toml` + `uv.lock`
- Lockfile: present (`uv.lock`)

## Frameworks

**Core ML:**
- TensorFlow 2.18.1 — backend compute engine; CPU-only (no CUDA drivers detected)
- Keras 3.6.0 — high-level model API; used directly via `tensorflow.keras`

**Training toolkit:**
- `keras.datasets.mnist` — built-in MNIST dataset loader
- `tf.keras.utils.image_dataset_from_directory` — loads the external validation set from a local directory
- `tensorflow.keras.callbacks.EarlyStopping` — training callbacks
- `tensorflow.keras.optimizers.Adam` — optimizer

**Notebook:**
- JupyterLab / Jupyter Notebook — `ipykernel` 7.2.0, `jupyter-client` 8.8.0, `jupyter-core` 5.9.1
- IPython 9.13.0 — kernel

**Visualization:**
- matplotlib 3.8+ (with `matplotlib-inline` 0.1.7) — loss curve plots rendered via `FigureCanvasAgg` (headless, no display required)
- Pillow 12.2.0 — image support (matplotlib backend dependency)

**Build/Dev:**
- UV — dependency resolution and virtual environment management
- debugpy 1.8.20 — notebook debugging support

## Key Dependencies

**Critical:**
- `tensorflow==2.18.*` — model construction, training, serialization
- `keras==3.6.*` — Sequential API, layer types, callbacks
- `numpy>=1.26,<2.2` — array operations; resolved to 2.0.2 in lockfile
- `requests>=2.32` — HTTP client; resolved to 2.33.1 (present in `pyproject.toml` but not actively used in current notebook cells)

**Infrastructure:**
- `h5py` 3.16.0 — HDF5 model weight serialization (Keras `.h5` saves)
- `grpcio` 1.80.0 — TensorBoard/gRPC transport
- `tensorboard` 2.18.0 — training metrics visualization (optional)
- `protobuf` 5.29.6 — TensorFlow serialization format
- `tensorflow-io-gcs-filesystem` 0.37.1 — GCS dataset access (bundled with TF)

## Configuration

**Environment:**
- No `.env` file present
- No environment variables required for local training; TensorFlow uses `TF_ENABLE_ONEDNN_OPTS` (optional, noted in logs)
- Notebook sets `MPLBACKEND=agg` and `MPLCONFIGDIR=/tmp/matplotlib-config` at runtime for headless rendering

**Build:**
- `pyproject.toml` — project metadata and direct dependencies
- `uv.lock` — fully pinned transitive dependency tree
- `[tool.uv] package = false` — project is not installed as a package; dependencies only

## Platform Requirements

**Development:**
- Python >=3.11, <3.13
- UV installed (`uv sync` installs all dependencies into `.venv/`)
- Jupyter-compatible environment (`jupyter notebook notebook.ipynb`)
- CPU-only execution confirmed; GPU support not available in current environment

**Production:**
- No server deployment; deliverable is `notebook.ipynb`
- Model weight file must be < 800 KB (grading constraint)
- Model must accept `(28, 28)` uint8 input and output `(10,)` softmax probabilities

---

*Stack analysis: 2026-05-21*
