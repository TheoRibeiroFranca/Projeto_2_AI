# Phase 4: Advanced Model Notebook + Polish - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions captured in CONTEXT.md — this log preserves the discussion.

**Date:** 2026-05-23
**Phase:** 04-advanced-model-notebook-polish
**Mode:** discuss (default)
**Areas discussed:** GBM variant, Hyperparameter tuning, Documentation depth

---

## Areas Discussed

### GBM variant
| Question | Options presented | User selection |
|----------|------------------|----------------|
| Which GBM class to use? | HistGradientBoostingClassifier (Recommended) / GradientBoostingClassifier + sample_weight | HistGradientBoostingClassifier (Recommended) |

**Notes:** `GradientBoostingClassifier` (named in REQUIREMENTS.md) doesn't support `class_weight='balanced'` natively. `HistGradientBoostingClassifier` is the modern sklearn replacement, supports it directly, and matches the API pattern of LR/RF notebooks. User confirmed the practical choice over the literal spec name.

---

### Hyperparameter tuning
| Question | Options presented | User selection |
|----------|------------------|----------------|
| Tuning approach? | Manual params with documented rationale (Recommended) / Light GridSearchCV on key params | Manual params with documented rationale (Recommended) |

**Notes:** Phase 3 context explicitly deferred GridSearchCV. User confirmed: manual parameter selection with inline comments explaining rationale. If 65-70% target isn't reached, report actual result honestly.

---

### Documentation depth (EVAL-04)
| Question | Options presented | User selection |
|----------|------------------|----------------|
| How much documentation to add? | Focused: intro + results cells only (Recommended) / Full prose: each section explained | Focused: intro + results cells only (Recommended) |

**Notes:** Applies to all three model notebooks. Decision: add two new cells per notebook — title/intro at top, results interpretation at bottom. Section headers (`## Load Data`, etc.) stay as-is. Model-specific rationale included in each intro cell.

---

## Deferred Ideas

- GridSearchCV/RandomizedSearchCV — explicitly out of scope for Phase 4
- Expanding documentation beyond two cells per notebook — not required for EVAL-04

---

## Claude's Discretion Items

- Exact HGBC parameter values (learning_rate, max_iter, max_depth, l2_regularization)
- Exact wording of intro and results interpretation cells
- random_state=42 for reproducibility (consistent with existing pattern)
