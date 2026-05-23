---
phase: 2
slug: feature-engineering
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-05-22
approved_date: 2026-05-23
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
| 02-01-01 | 01 | 0 | — | — | N/A | env check | `uv sync && python -c "import sklearn"` | ✅ | ✅ green |
| 02-01-02 | 01 | 1 | FEAT-01 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |
| 02-01-03 | 01 | 1 | FEAT-02 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |
| 02-01-04 | 01 | 1 | FEAT-03 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |
| 02-01-05 | 01 | 1 | FEAT-04 | — | N/A | assertion in notebook | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |
| 02-02-01 | 02 | 2 | FEAT-01–04 | — | N/A | integration + leakage | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [x] `uv sync` — scikit-learn 1.8.0 installed in venv (scikit-learn>=1.3 in pyproject.toml; confirmed installed via `uv sync` in Plan 01 Task 1)
- [x] Confirm `pytest --nbmake notebook_data.ipynb -x` passes on Phase 1 cells before adding Phase 2 cells — confirmed green after Plan 01 Task 2 nbconvert step

uv sync executed in Plan 01 Task 1; pytest --nbmake confirmed green after Plan 01 Task 2 and again after Plan 02 Task 1. Existing infrastructure now covers all Phase 2 requirements.

---

## Manual-Only Verifications

All Phase 2 behaviors have automated verification via assertions inside notebook_data.ipynb plus pytest --nbmake. The RESEARCH.md-suggested manual spot-check (sample rows for one team-season) is satisfied by the section 12 groupby('result').mean() output which is captured in the executed .ipynb.

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 30s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-05-23
