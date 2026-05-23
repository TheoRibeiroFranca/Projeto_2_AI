---
phase: 02-feature-engineering
fixed_at: 2026-05-23T13:00:00Z
review_path: .planning/phases/02-feature-engineering/02-REVIEW.md
iteration: 1
findings_in_scope: 6
fixed: 6
skipped: 0
status: all_fixed
---

# Phase 02: Code Review Fix Report

**Fixed at:** 2026-05-23T13:00:00Z
**Source review:** .planning/phases/02-feature-engineering/02-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 6 (CR-01, CR-02, WR-01, WR-02, WR-03, WR-04; IN-01 and IN-02 excluded per fix_scope=critical_warning)
- Fixed: 6
- Skipped: 0

## Fixed Issues

### CR-01: `derive_result` else-clause silently misclassifies unexpected `vencedor` values as AwayWin

**Files modified:** `notebook_data.ipynb`
**Commit:** 46dbdb2
**Applied fix:** Added an explicit `elif row['vencedor'] == row['visitante']: return 'AwayWin'` branch and replaced the silent `else: return 'AwayWin'` fallthrough with `raise ValueError(...)` that includes the match ID, mandante, and visitante in the error message. Any future data corruption or name variant that does not match either team will now raise immediately instead of being silently mislabelled.

---

### CR-02: Same-day double-header in long-format table creates rolling-window leakage for 31 team-match pairs

**Files modified:** `notebook_data.ipynb`
**Commit:** e7e5fae
**Applied fix:** Added `'round'` as a tie-break key in `sort_values(['team', 'season', 'round', 'date', 'id'])` in `build_features` Step 2, stabilizing same-day match ordering by round number rather than arbitrary `id` order. Also added a diagnostic warning block that detects and prints `(team, date)` pairs appearing twice in the long-format table (rescheduled fixture artifacts), so users are informed of the condition without breaking execution.

---

### WR-01: `derive_result` executes before team name normalization — design ordering risk

**Files modified:** `notebook_data.ipynb`
**Commit:** 9e50437
**Applied fix:** Moved the `canonical_names = {}` definition into cell `3391c415` (before `derive_result`) and applied the same normalization dict to `mandante`, `visitante`, and `vencedor` before calling `derive_result`. Updated cell `e2420af0` (Section 6) to remove the now-redundant `canonical_names = {}` definition and updated its comment to clarify it applies the already-defined dict to the renamed `home_team`/`away_team` columns.

---

### WR-02: `points_last5` and `goal_diff_last5` computed before imputation loop is dead code

**Files modified:** `notebook_data.ipynb`
**Commit:** 6339925
**Applied fix:** Removed the pre-imputation `FEAT-04` block (3 lines: comment + 2 assignments) that computed `points_last5` and `goal_diff_last5` before the imputation loop. These NaN-polluted values were immediately overwritten by the correct post-imputation re-derivation which is retained unchanged.

---

### WR-03: `is_monotonic_increasing` assertion is imprecise for the leakage it is intended to guard

**Files modified:** `notebook_data.ipynb`
**Commit:** 559a78a
**Applied fix:** Replaced the misleading single-line assertion comment ("CSV não está ordenado por data — ordenação necessária") with a two-line comment accurately stating that the assertion checks non-decreasing date order (equal dates are allowed for same-day matches) and that `build_features` re-sorts by `(date, id)` to stabilize same-day order. Updated the assertion message to be more descriptive about what a failure indicates.

---

### WR-04: No assertion validates the `win + draw + loss` rolling sum identity

**Files modified:** `notebook_data.ipynb`
**Commit:** 1de3a85
**Applied fix:** Added a W+D+L rolling sum identity assertion inside `build_features` after the existing NaN assertion, checking that `home_wins_last5 + home_draws_last5 + home_losses_last5` equals exactly 5 (full window) or 0 (imputed rows for newly-promoted teams with no prior season data). Any future imputation scale error that produces non-5/non-0 sums will be caught immediately.

---

_Fixed: 2026-05-23T13:00:00Z_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
