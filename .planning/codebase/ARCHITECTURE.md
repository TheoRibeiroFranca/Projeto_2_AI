<!-- refreshed: 2026-05-21 -->
# Architecture

**Analysis Date:** 2026-05-21

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│                    notebook.ipynb                           │
│            (single-file deliverable, 4 sections)           │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│                    Data Layer                                │
│   MNIST (keras.datasets) + external validation-set.zip      │
│   Augmented: x_train_inv = 255 - x_train                    │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│                    Model Layer                               │
│   Keras Sequential: Rescaling → Flatten → Dense blocks      │
│   Each block: Dense → BatchNormalization → Dropout           │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│             Two-Phase Training Orchestration                 │
│  Phase 1: Adam lr=0.0005, bs=256, ≤10 epochs               │
│  Phase 2: Adam lr=0.00005, bs=16, ≤40 epochs               │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│             Evaluation & Visualization                       │
│   model.evaluate on x_test_inv, combined loss curve plot    │
└─────────────────────────────────────────────────────────────┘
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

**Overall:** Linear Jupyter Notebook pipeline — single-file ML experiment with sequential cell execution.

**Key Characteristics:**
- All logic lives in `notebook.ipynb`; there are no importable Python modules
- Preprocessing is baked into the model as `Rescaling` and `Flatten` layers (grading requirement)
- Two-phase training separates broad generalization (Phase 1) from domain-specific fine-tuning (Phase 2)
- Data augmentation uses pixel inversion to simulate the external validation set's domain

## Layers

**Data Ingestion:**
- Purpose: Load, augment, and concatenate training data
- Location: `notebook.ipynb` cells `8b3f3fbb` and `35d075b9`
- Contains: MNIST loading, zip extraction, numpy array assembly, pixel inversion
- Depends on: `keras.datasets.mnist`, `zipfile`, `tensorflow.keras.utils.image_dataset_from_directory`
- Used by: Training orchestration layer

**Model Definition:**
- Purpose: Declare the neural network architecture as a Keras Sequential model
- Location: `notebook.ipynb` cell `7a84b337`
- Contains: `Rescaling(1/255)`, `Flatten`, four `Dense→BatchNormalization→Dropout` blocks, softmax output
- Allowed layers only: `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`
- Depends on: `tensorflow.keras`
- Used by: Training orchestration layer

**Training Orchestration:**
- Purpose: Execute two-phase training with separate hyperparameters and data distributions
- Location: `notebook.ipynb` cell `35d075b9`
- Contains: `model.fit` calls for Phase 1 and Phase 2, EarlyStopping callbacks, data tiling logic
- Depends on: Model Definition layer, Data Ingestion layer
- Used by: Evaluation layer

**Evaluation & Visualization:**
- Purpose: Report accuracy and render loss curves
- Location: `notebook.ipynb` cells `3fd76014` and `0967d16e`
- Contains: `model.evaluate`, matplotlib figure combining both phase histories
- Depends on: Trained model, history objects from both phases
- Used by: Grader (notebook output)

## Data Flow

### Primary Training Path

1. Load MNIST from `keras.datasets` — `(x_train, y_train), (x_test, y_test)` as uint8 (notebook cell `8b3f3fbb`)
2. Extract `validation-set.zip` to `./validation-set/digits/`, load via `image_dataset_from_directory` → `x_extra`, `y_extra` (cell `35d075b9`)
3. Invert training pixels: `x_train_inv = 255 - x_train` (cell `35d075b9`)
4. Concatenate Phase 1 dataset: `[x_train_inv, x_extra, x_sint]` → `x_train_final` (cell `35d075b9`)
5. Phase 1 `model.fit(x_train_final, ...)` — 10 epochs max, Adam lr=0.0005, bs=256 (cell `35d075b9`)
6. Tile external data 10x: `x_extra_rep2 = np.tile(x_extra, (10,1,1))` (cell `35d075b9`)
7. Phase 2 `model.fit(x_extra_rep2, ...)` — 40 epochs max, Adam lr=0.00005, bs=16 (cell `35d075b9`)

### Inference / Evaluation Path

1. `model.evaluate(x_train_final, y_train_final)` → train accuracy (cell `3fd76014`)
2. `model.evaluate(x_test_inv, y_test)` → test accuracy (cell `3fd76014`)
3. Combine `history_fase1.history` and `history_fase2.history` loss lists, plot with vertical separator at phase boundary (cell `0967d16e`)

**State Management:**
- Model weights are held in-memory in the `model` variable throughout the notebook session
- Training history is stored in `history_fase1` and `history_fase2` dict-like objects
- No model serialization (saving/loading weights) is present in the current notebook

## Key Abstractions

**Keras Sequential Model:**
- Purpose: Encapsulates the full inference pipeline including preprocessing
- Location: `notebook.ipynb` cell `7a84b337`
- Pattern: `model = Sequential([Rescaling(1/255), Flatten(), Dense(200, 'relu'), BatchNormalization(), Dropout(0.1), ...])`

**Two-Phase History Merging:**
- Purpose: Produce a single continuous loss curve across both training phases
- Location: `notebook.ipynb` cell `0967d16e`
- Pattern: Concatenate `history_fase1.history['loss'] + history_fase2.history['loss']`, draw `axvline` at `fim_fase1`

**Pixel Inversion Augmentation:**
- Purpose: Domain adaptation — external validation set uses dark-on-light images; MNIST uses light-on-dark
- Location: `notebook.ipynb` cell `35d075b9`
- Pattern: `x_train_inv = 255 - x_train`

## Entry Points

**Notebook Execution:**
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

**What happens:** Cell `35d075b9` concatenates `x_sint` and `y_sint` into `x_train_final` but no cell in the notebook defines these variables.
**Why it's wrong:** Running the notebook top-to-bottom will raise a `NameError` at Phase 1 training.
**Do this instead:** Either define `x_sint`/`y_sint` in a preceding cell, or remove them from the concatenation.

### No Model Persistence

**What happens:** The trained model is never saved with `model.save()` or `model.save_weights()`.
**Why it's wrong:** Closing the kernel loses all trained weights; the model cannot be submitted or reloaded without retraining.
**Do this instead:** Add `model.save('model.keras')` after Phase 2 training completes.

### Final Dense Layer Without Softmax Activation

**What happens:** The output `Dense(10)` layer in cell `7a84b337` has no activation specified (defaults to linear), while the loss `sparse_categorical_crossentropy` with `from_logits=False` expects probabilities.
**Why it's wrong:** Produces logits instead of softmax probabilities, violating the output contract `(10,)` softmax.
**Do this instead:** Use `Dense(10, activation='softmax')`.

## Error Handling

**Strategy:** None — the notebook has no try/except blocks. Errors propagate as cell exceptions and halt execution.

**Patterns:**
- No explicit error handling around zip extraction or HTTP data loading
- No input validation on external dataset shape before concatenation

## Cross-Cutting Concerns

**Logging:** TensorFlow/Keras training progress printed automatically via `model.fit` verbosity (default=1)
**Validation:** Keras enforces layer constraints at `model.compile` / `model.fit` time; no custom validation
**Authentication:** Not applicable — external data loaded from local zip file, no API keys required

---

*Architecture analysis: 2026-05-21*
