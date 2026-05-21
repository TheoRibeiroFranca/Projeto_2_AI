# Testing

*Last mapped: 2026-05-21*

## Summary

No formal test framework exists in this codebase. Verification is interactive and constraint-based rather than automated.

## Test Framework

**None installed.** No pytest, unittest, or any other test framework is present. There are no test files (`test_*.py`, `*_test.py`, `tests/` directory, or similar).

## Verification Approach

Testing is done through two mechanisms:

### 1. Interactive Notebook Diagnostics
- `print()` statements in `notebook.ipynb` cells display training metrics
- Loss curve plots visualize training/validation progress across epochs
- Outputs are inspected manually during development

### 2. External Grading Server (Model Contract)
The model is submitted to an external grading server that enforces hard constraints:
- Input shape: `(28, 28)` uint8 images
- Output shape: `(10,)` softmax probabilities
- Weight file size: < 800 KB
- Allowed layers only: `Dense`, `Flatten`, `Rescaling`, `BatchNormalization`, `Dropout`

Passing these constraints is determined externally, not locally.

## Coverage

- No unit tests
- No integration tests
- No automated accuracy benchmarks
- No CI/CD pipeline

## Recommendations (if tests are added)

- Use `pytest` with `numpy` assertions for model output shape/range checks
- Add a weight-size check: `assert model_size_kb < 800`
- Validate softmax outputs sum to 1.0 per sample
- Smoke-test on a small MNIST batch to confirm forward pass works

---
*Mapped: 2026-05-21*
