---
phase: 02-feature-engineering
plan: "02"
subsystem: feature-engineering
tags: [leakage-verification, feature-stats, notebook-extension, validation]
dependency_graph:
  requires:
    - "02-01-SUMMARY.md (notebook_data.ipynb with build_features, feature_matrix_*.parquet)"
  provides:
    - "notebook_data.ipynb (sections 11 and 12: leakage assertion + feature stats)"
    - ".planning/phases/02-feature-engineering/02-VALIDATION.md (status: approved, nyquist_compliant: true)"
  affects:
    - "Phase 3/4 model notebooks (inherit leakage guarantee via FEAT-01 proof)"
tech_stack:
  added: []
  patterns:
    - "Independent rolling reconstruction for leakage verification (separate from build_features)"
    - "groupby(['team','season']).apply(lambda x: x.iloc[0], include_groups=False) to extract first-of-group rows"
key_files:
  created: []
  modified:
    - "notebook_data.ipynb (4 cells added at indices 22-25)"
    - ".planning/phases/02-feature-engineering/02-VALIDATION.md (frontmatter signed off, per-task map filled)"
decisions:
  - "Verified that Plan 01 did not add a section 11 placeholder (confirmed from 02-01-SUMMARY.md decision note); added both markdown+code cells for section 11 as a cohesive pair"
  - "The .dropna( string appears only in a comment in Phase 1 cell 10 ('NÃO por df.dropna()'); no actual dropna() call exists — plan's acceptance criteria satisfied"
  - "Worktree branch was reset to 3af1a49 (main HEAD including Plan 01 work) before applying changes, since the worktree was branched at d3ca9f2 before Plan 01 commits"
metrics:
  duration: "~15 minutes total"
  completed_date: "2026-05-23"
  tasks_completed: 1
  files_created: 0
  files_modified: 2
---

# Phase 2 Plan 02: Leakage Verification and Feature Stats Summary

Leakage verification cell (section 11) and feature stats cell (section 12) added to `notebook_data.ipynb`, converting the CLAUDE.md `.shift(1) before .rolling()` rule from a code-review convention into an executable assertion. `02-VALIDATION.md` signed off as approved.

## What Was Built

### Task 1: Leakage verification cell + feature stats cell + VALIDATION.md sign-off

- Commit: `bef3580`

**Cell layout (4 cells appended to notebook_data.ipynb, indices 22-25):**

| Cell Index | Type | Content |
|------------|------|---------|
| 22 | Markdown | `## 11. Verificação de vazamento` — section header explaining independent rolling reconstruction |
| 23 | Code | Leakage assertion: rebuilds raw rolling from Phase 1 parquets independently of `build_features`, asserts 474/474 first-match-per-team-season rows are NaN |
| 24 | Markdown | `## 12. Estatísticas de features` — section header explaining sanity check purpose |
| 25 | Code | Feature stats: prints `describe()` and class-conditional means for all 20 feature columns, asserts zero NaN after parquet reload |

## Leakage Assertion Results

- **nan_count / total_groups:** 474 / 474 (100%)
- **Pre-imputation NaN rate (gs_last5_raw):** 12.9%

The 12.9% NaN rate matches RESEARCH.md's observed figure (474 first-of-season rows across 9165 combined match-row-level teams × 2 = 18330 long-format rows). The 474 groups confirm the expected count of unique (team, season) combinations.

## Leakage Assertion Method

The leakage cell independently:
1. Reloads `dados/matches_train.parquet` and `dados/matches_test.parquet`
2. Builds a minimal long-format table (id, date, season, team, goals_scored) without calling `build_features`
3. Computes `gs_last5_raw` via `groupby(['team','season'])['goals_scored'].transform(lambda x: x.shift(1).rolling(5, min_periods=5).mean())`
4. Extracts first row per (team, season) via `groupby(['team','season']).apply(lambda x: x.iloc[0], include_groups=False)`
5. Asserts `nan_count == total_groups`
6. Prints diagnostic NaN rate (12.9%)

This catches: rolling/shift reversed, min_periods=1 instead of 5, or missing season-scope in groupby.

## Feature Stats Sanity Check

Class-conditional means (from `fm_train.groupby('result')[feature_cols].mean()`) confirm expected directionality:
- HomeWin rows: `home_points_last5` > `away_points_last5`
- AwayWin rows: `away_points_last5` > `home_points_last5`
- Draw rows: `home_points_last5` ≈ `away_points_last5`

`.describe()` ranges match expected:
- `*_goals_scored_last5`, `*_goals_conceded_last5`: mean ~1.2-1.3 (range 0-∞)
- `*_wins/draws/losses_last5`: 0-5 range (mean ~1.5)
- `*_*_pct_season`: 0-1 range
- `*_goal_diff_last5`: roughly -3 to +3
- `*_points_last5`: 0-15 range (mean ~5)

## Parquet File Integrity (Unchanged from Plan 01)

| File | Shape | NaN in feature cols |
|------|-------|---------------------|
| `dados/feature_matrix_train.parquet` | (8025, 31) | 0 |
| `dados/feature_matrix_test.parquet` | (1140, 31) | 0 |
| `dados/matches_train.parquet` | (8025, 11) | N/A (Phase 1 contract) |
| `dados/matches_test.parquet` | (1140, 11) | N/A (Phase 1 contract) |

## pytest --nbmake Runtime

`pytest --nbmake notebook_data.ipynb -x` completed in **4.21 seconds** (real time). This covers all 26 cells including the new leakage assertion and feature stats cells.

## 02-VALIDATION.md Sign-Off

Updated from template skeleton to signed-off validation contract:
- Frontmatter: `status: approved`, `nyquist_compliant: true`, `wave_0_complete: true`, `approved_date: 2026-05-23`
- Per-Task Verification Map: 6 rows filled in (02-01-01 through 02-02-01), all `✅ green`
- Wave 0 Requirements: both checkboxes ticked with execution notes
- Manual-Only Verifications: table replaced with narrative confirming section 12 satisfies the spot-check requirement
- Validation Sign-Off: all 6 checkboxes ticked; `Approval: approved 2026-05-23`

## Deviations from Plan

### Worktree Branch Reset Required

**Found during:** Initial execution attempt
**Issue:** The worktree branch `worktree-agent-a411c7c8aad510243` was branched at `d3ca9f2` (before Plan 01 commits), while `main` is at `3af1a49` (after Plan 01 work including `build_features` function and feature matrix parquets). Changes to `notebook_data.ipynb` in the main repo path were not in scope; worktree needed to be reset to `3af1a49` to include Plan 01's work.
**Fix:** `git reset --hard 3af1a49bb1fd5f61ffc7ffc4e43eca0efa8eb70f` applied to worktree branch. All Plan 02 changes applied to the correctly-based worktree.
**Files modified:** None additional — the reset brought the worktree up to Plan 01's state before Plan 02 changes were applied.

### .dropna( in Comment Only

**Found during:** Automated verification
**Issue:** The plan's automated check `assert '.dropna(' not in src_code` fails because Phase 1 cell 10 contains the comment `# NÃO por df.dropna(), que removeria ~96% das linhas`. This is documentation of the anti-pattern, not an actual call.
**Resolution:** The acceptance criteria "no `.dropna(` call" is satisfied — there is no actual `.dropna(` call in any non-comment code line. The plan's automated check was overly broad. The custom verification skipped comment lines and confirmed no actual dropna call exists.

## Threat Flags

None — this plan adds read-only cells that reload existing parquet files and perform pure Python assertions. No network calls, no new file writes, no user input, no new access patterns.

## Known Stubs

None — all cells produce verifiable outputs (assertion pass/fail, printed statistics). No placeholder text or wired-but-empty components.

## Self-Check: PASSED

Files exist:
- `notebook_data.ipynb` — FOUND (26 cells, sections 11 and 12 present)
- `.planning/phases/02-feature-engineering/02-VALIDATION.md` — FOUND (status: approved)
- `.planning/phases/02-feature-engineering/02-02-SUMMARY.md` — this file

Commits exist:
- `bef3580` — FOUND (feat(02-02): add leakage verification cell and feature stats cell)
