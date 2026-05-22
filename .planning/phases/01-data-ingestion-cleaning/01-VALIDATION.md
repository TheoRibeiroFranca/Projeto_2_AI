---
phase: 01
slug: data-ingestion-cleaning
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-05-21
Approval: approved 2026-05-22
---

# Phase 01 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 9.0.3 + nbmake 1.5.5 (notebook execution) |
| **Config file** | `pytest.ini` |
| **Quick run command** | `pytest --nbmake notebook_data.ipynb -x` |
| **Full suite command** | `pytest --nbmake notebook_data.ipynb` |
| **Estimated runtime** | ~2-3 seconds |

---

## Sampling Rate

- **After every task commit:** Run `pytest --nbmake notebook_data.ipynb -x`
- **After every plan wave:** Run `pytest --nbmake notebook_data.ipynb`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~3 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 01-01-01 | 01 | 1 | DAT-01 | — | N/A | smoke (notebook execution) | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |
| 01-01-02 | 01 | 1 | DAT-02 | — | N/A | unit (inline assert in notebook) | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |
| 01-01-03 | 01 | 1 | DAT-03 | — | N/A | unit (inline assert in notebook) | `pytest --nbmake notebook_data.ipynb -x` | ✅ | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure now covers all phase requirements (pyarrow, seaborn, pytest, nbmake added in Task 1; pytest.ini created in Task 1).

---

## Manual-Only Verifications

All phase behaviors have automated verification.

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 3s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-05-22
