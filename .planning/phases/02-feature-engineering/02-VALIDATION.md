---
phase: 2
slug: feature-engineering
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-05-22
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 9.x + nbmake 1.5.x |
| **Config file** | none — run from command line |
| **Quick run command** | `pytest --nbmake notebook_data.ipynb -x` |
| **Full suite command** | `pytest --nbmake notebook_data.ipynb -x` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `pytest --nbmake notebook_data.ipynb -x`
- **After every plan wave:** Run `pytest --nbmake notebook_data.ipynb -x`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 02-01-01 | 01 | 0 | — | — | N/A | env check | `uv sync && python -c "import sklearn"` | ✅ | ⬜ pending |
| 02-01-02 | 01 | 1 | FEAT-01 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ⬜ pending |
| 02-01-03 | 01 | 1 | FEAT-02 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ⬜ pending |
| 02-01-04 | 01 | 1 | FEAT-03 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ⬜ pending |
| 02-01-05 | 01 | 1 | FEAT-04 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ⬜ pending |
| 02-01-06 | 01 | 2 | FEAT-01–04 | — | N/A | integration | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `uv sync` — ensure scikit-learn 1.7.x is installed in venv (needed for Phase 3; verify now to avoid surprises)
- [ ] Confirm `pytest --nbmake notebook_data.ipynb -x` passes on Phase 1 cells before adding Phase 2 cells

*If both already pass: existing infrastructure is sufficient.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Visual spot-check of feature values for a known team/season | FEAT-01–04 | Sanity check that values are numerically reasonable | Print sample rows for Flamengo 2022 and confirm goals_scored_last5 tracks match history |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
