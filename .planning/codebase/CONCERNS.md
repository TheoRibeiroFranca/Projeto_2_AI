# Concerns

*Last mapped: 2026-05-21*

## Critical Bugs

### Undefined variables `x_sint` / `y_sint`
- **Cell:** `35d075b9`
- **Impact:** `NameError` on fresh kernel run — notebook cannot execute end-to-end
- `np.concatenate([x_train_inv, x_extra, x_sint], ...)` references variables never defined

### Source/output mismatch on final Dense layer
- **Cell:** `7a84b337`
- **Impact:** Source shows `Dense(3, activation='softmax')` (3-class) but saved output shows `dense_4 (None, 10)` (10-class)
- Notebook was edited after running without re-executing — grader reading source sees wrong architecture

### `Rescaling` and `Flatten` missing from model source
- **Cell:** `7a84b337`
- **Impact:** Source omits both layers from `Sequential([...])` constructor, but saved summary shows them
- Cannot satisfy grading contract (uint8 input requires `Rescaling`) if re-run from source

### `validation-set.zip` not downloaded by code
- **Cell:** `35d075b9`
- **Impact:** Opens `"validation-set.zip"` via `zipfile` with no HTTP download, no fallback
- Any fresh clone or re-run fails at this cell

---

## Tech Debt

| Item | Location | Severity |
|------|----------|----------|
| `# TODO: train your model` and `# TODO: evaluate` left in implemented cells | Cells `35d075b9`, `3fd76014` | Medium — grading weights documentation quality |
| `from tensorflow.keras import regularizers` imported but never used | Cell `7a84b337` | Low |
| `x_extra_rep_fase1` / `y_extra_rep_fase1` computed but never used in `model.fit` | Cell `35d075b9` | Low — dead code from abandoned iteration |
| No `.gitignore` — `.venv/` (2 GB), `.ipynb_checkpoints/`, `__pycache__/` unguarded | Repo root | Medium |
| Kernel metadata declares wrong project: `"projeto1-mnist-starter (3.12.13)"` | `notebook.ipynb` metadata | Low |
| Markdown sections "2. Build your model" and "3. Train" are template boilerplate | Notebook | Medium — grading considers documentation |
| No `model.save()` anywhere — weights exist only in memory | Notebook | High — kernel crash loses all training |

---

## Performance / Fragile Areas

- **Phase 2 overfitting:** `val_accuracy` hits 1.0000 by epoch 2 but trains 15 more patience epochs — overfitting a 320-sample tile
- **Phase 1 plateau:** `val_accuracy` ~83.6% across all 10 epochs — domain gap from inverted training vs. normal-polarity validation data
- **Relative paths:** `"validation-set.zip"` and `"validation-set/digits"` assume CWD is repo root — fragile in notebooks
- **Weight size headroom:** 723.38 KB / 800 KB limit — only ~77 KB (~10%) headroom, no guard assertion

---

## Missing Critical Features

- No model contract validation cell — input shape, output shape, allowed layers, weight size only caught at grading time
- No held-out evaluation on original `x_extra` (pre-tiling); test accuracy measured on inverted MNIST, not external validation distribution

---

## Legacy Dead Data

- `dados/archive/` — ~4.6 MB of Brazilian football CSVs, unrelated to MNIST project
- `dados/archive.zip` — 692 KB, same legacy data
- Both can be safely removed

---

## Security

- No credentials or API keys detected in code
- HTTP fetch of external validation set uses a hardcoded URL — URL change would break training silently

---
*Mapped: 2026-05-21*
