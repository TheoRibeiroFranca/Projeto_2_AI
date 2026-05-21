# External Integrations

**Analysis Date:** 2026-05-21

## APIs & External Services

**Built-in Dataset:**
- Keras MNIST — training and test data loaded via `keras.datasets.mnist.load_data()`
  - SDK/Client: `tensorflow.keras` (bundled)
  - Auth: None; data cached locally by Keras on first load (`~/.keras/datasets/`)

**External Validation Set:**
- Local zip archive — `validation-set.zip` extracted at training time via `zipfile.ZipFile`
  - Loaded with: `tf.keras.utils.image_dataset_from_directory("validation-set/digits", ...)`
  - Format: directory of 28x28 grayscale images, 10 class subdirectories, 320 total images
  - Auth: None; file must be present in the working directory before training cells run
  - Note: CLAUDE.md references an HTTP URL for this file, but current notebook code reads it from a local zip — no active HTTP call exists in committed notebook cells

## Data Storage

**Datasets:**
- MNIST — auto-downloaded and cached by Keras; stored in `~/.keras/datasets/mnist.npz`
- External validation set — `validation-set.zip` in the project root (not committed to git; must be present at runtime)
- Legacy CSV data — `dados/archive/` contains Brazilian football championship CSVs; not used by `notebook.ipynb`

**File Storage:**
- Local filesystem only; no cloud object storage

**Caching:**
- Keras dataset cache at `~/.keras/datasets/` (managed by Keras automatically)
- No application-level cache

## Authentication & Identity

**Auth Provider:**
- None — project has no user-facing auth; no login, tokens, or API keys required

## Monitoring & Observability

**Error Tracking:**
- None

**Logs:**
- TensorFlow runtime logs printed to stderr (training progress, CUDA warnings)
- Matplotlib configured to write to `/tmp/matplotlib-config` at runtime
- TensorBoard 2.18.0 is installed as a TF dependency but no `TensorBoard` callback is configured in the notebook

## CI/CD & Deployment

**Hosting:**
- Not applicable — deliverable is a Jupyter notebook, not a deployed service

**CI Pipeline:**
- None detected

## Environment Configuration

**Required env vars:**
- None required; all configuration is in-notebook
- Optional: `TF_ENABLE_ONEDNN_OPTS=0` to suppress oneDNN log noise
- Runtime-set: `MPLBACKEND=agg`, `MPLCONFIGDIR=/tmp/matplotlib-config` (set inside notebook cell before plotting)

**Secrets location:**
- No secrets required; no `.env` file present

## Webhooks & Callbacks

**Incoming:**
- None

**Outgoing:**
- None; `requests` library is declared as a dependency but no outbound HTTP calls are present in committed notebook code

---

*Integration audit: 2026-05-21*
