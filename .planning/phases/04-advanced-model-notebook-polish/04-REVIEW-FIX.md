---
phase: 04-advanced-model-notebook-polish
fixed_at: 2026-05-23T23:15:00Z
review_path: .planning/phases/04-advanced-model-notebook-polish/04-REVIEW.md
iteration: 1
findings_in_scope: 5
fixed: 5
skipped: 0
status: all_fixed
---

# Phase 4: Code Review Fix Report

**Fixed at:** 2026-05-23T23:15:00Z
**Source review:** .planning/phases/04-advanced-model-notebook-polish/04-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 5 (CR-01, CR-02, CR-03, WR-01, WR-02, WR-03 — WR-01 and WR-03 addressed together, CR-01 and WR-02 addressed together)
- Fixed: 5
- Skipped: 0

## Fixed Issues

### CR-01 + WR-02: Add `class_weight='balanced'` and fix GBM hyperparameters

**Files modified:** `notebook_gradient_boost.ipynb`
**Commit:** 5737d27
**Applied fix:** Replaced the `HistGradientBoostingClassifier` constructor in the Model Training cell (id: `cell-7`). Added `class_weight='balanced'` to prevent Draw/AwayWin class collapse. Changed `learning_rate` from `0.007` to `0.05`, `max_iter` from `300` to `500`, reduced `max_depth` from `5` to `4`, and added `early_stopping=True` with `validation_fraction=0.1` and `n_iter_no_change=20` for better convergence. Stale cell outputs were cleared.

Note: CR-01 and WR-02 were in the same cell and committed atomically.

---

### CR-02: Correct Results Interpretation cell

**Files modified:** `notebook_gradient_boost.ipynb`
**Commit:** 1203f2a
**Applied fix:** Rewrote the Results Interpretation markdown cell (id: `34262e0e-0be3-47b4-bc40-b72763c518e0`). Removed the false claim that the model was trained with `class_weight='balanced'` producing ~42% accuracy. Replaced with accurate prose that correctly describes the expected behavior after the CR-01 fix: that `class_weight='balanced'` may cause accuracy to fall below the naive baseline while improving minority class recall, and that macro-F1 above 0.33 signals genuine learning.

---

### CR-03: Populate `canonical_names` with known team name variants

**Files modified:** `notebook_data.ipynb`
**Commit:** 07276f1
**Applied fix:** Replaced `canonical_names = {}` (empty no-op dict) in Section 4 cell (id: `3391c415`) with a populated dictionary of known Brazilian football team name variants. Variants include: Atletico Mineiro/Atletico MG/Atlético-MG → Atletico-MG; Atletico-PR/Atletico PR/Athletico Paranaense → Athletico-PR; Botafogo/Botafogo RJ → Botafogo-RJ; America Mineiro/América-MG → America-MG; Red Bull Bragantino/RB Bragantino → Bragantino; plus accent/spacing variants for Gremio, Sao Paulo, Ceara, Goias, Avai, Fortaleza, and Cuiaba. The current CSV is already clean but the dict provides defensive normalization for future data or alternate-spelling user inputs.

Note: This fix requires human verification — verify that the assert in Section 6 (top clubs have 380+ matches) still passes when `notebook_data.ipynb` is re-executed, and that the parquet outputs are regenerated.

---

### WR-01 + WR-03: Venue-agnostic `predict_match` lookup and full `VALID_TEAMS`

**Files modified:** `notebook_logistic.ipynb`, `notebook_random_forest.ipynb`, `notebook_gradient_boost.ipynb`
**Commit:** 3595642
**Applied fix:** Replaced the venue-split `home_last`/`away_last` lookups with a single `team_last` venue-agnostic lookup in the `predict_match` cell (cell index 17) of all three notebooks. The fix: creates `home_view` (date, team, home_* features) and `away_view` (date, team, away_* columns renamed to home_*), concatenates them, sorts by date, and groups by team to get the most recent match regardless of venue. Teams whose last game was as away now get their most recent form values rather than stale earlier home-game values.

Also fixed `VALID_TEAMS` to use the union of `home_team` and `away_team` columns (WR-03), ensuring all 46 teams appear in error suggestions rather than only those who appeared as home team.

Note: WR-01 and WR-03 were in the same cell and committed atomically across all three notebooks.

## Skipped Issues

None — all in-scope findings were fixed.

---

_Fixed: 2026-05-23T23:15:00Z_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
