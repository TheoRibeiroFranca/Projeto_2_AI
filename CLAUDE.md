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
