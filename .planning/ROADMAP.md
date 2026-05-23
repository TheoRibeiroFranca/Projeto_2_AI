# Roadmap: Brazilian League Match Predictor

**Milestone:** v1
**Granularity:** Standard
**Total phases:** 4
**Requirements covered:** 15 / 15

---

## Phases

- [x] **Phase 1: Data Ingestion & Cleaning** - Load CSVs, normalize team names, and apply temporal split so downstream work has a clean, leak-free dataset (completed 2026-05-22)
- [ ] **Phase 2: Feature Engineering** - Compute all rolling-window and form features with strict temporal integrity (no leakage)
- [ ] **Phase 3: Baseline Model Notebooks** - Build and evaluate Logistic Regression and Random Forest classifiers with full evaluation suite
- [ ] **Phase 4: Advanced Model Notebook + Polish** - Deliver Gradient Boosting notebook targeting 65-70% accuracy with documentation quality for submission

---

## Phase Details

### Phase 1: Data Ingestion & Cleaning

**Goal**: A clean, temporally-ordered `matches` DataFrame exists with canonical team names and a fixed train/test split boundary
**Depends on**: Nothing (first phase)
**Requirements**: DAT-01, DAT-02, DAT-03
**Success Criteria** (what must be TRUE):

  1. `campeonato-brasileiro-full.csv` loads without error and all 9,165 rows are accounted for
  2. Team name normalization maps every variant to a canonical form — top clubs each have 380+ match rows after normalization
  3. Data is sorted by date and a fixed year cutoff produces non-overlapping train and test sets (no row appears in both)
  4. An EDA cell confirms the observed home win rate and class distribution (Home Win / Draw / Away Win) for the full dataset

**Plans:** 1/1 plans complete
Plans:

- [x] 01-01-PLAN.md — Add notebook deps (pyarrow, seaborn, pytest, nbmake), build `notebook_data.ipynb` (load → parse → derive → normalize → rename → split → save), and lock end-to-end nbmake validation with EDA

### Phase 2: Feature Engineering

**Goal**: A `feature_matrix` DataFrame exists where every row is a match with pre-match-only features and no data leakage
**Depends on**: Phase 1
**Requirements**: FEAT-01, FEAT-02, FEAT-03, FEAT-04
**Success Criteria** (what must be TRUE):

  1. Rolling 5-match goals scored and conceded columns are present; round-1 rows for each team show NaN (confirming `.shift(1)` was applied before `.rolling(5)`)
  2. W/D/L form streak columns (last 5 matches) are present for both home and away team
  3. Home win%, draw%, loss% and away win%, draw%, loss% columns are present and computed per team per current season
  4. Derived columns goal_difference_last5 and points_last5 are present and consistent with the underlying goals/result columns
  5. A reusable `build_features(df)` function exists that can reconstruct the feature matrix from any filtered date range without modification

**Plans:** 1/2 plans executed
Plans:
**Wave 1**

- [x] 02-01-PLAN.md — `uv sync` + define `build_features(df)` in notebook_data.ipynb (FEAT-01/02/03/04 in one function) + apply on combined train+test, re-split by season, save `dados/feature_matrix_{train,test}.parquet` (8025×31 and 1140×31, 0 NaN)

**Wave 2** *(blocked on Wave 1 completion)*

- [ ] 02-02-PLAN.md — Add leakage verification cell (raw shift(1).rolling(5) reconstruction asserts first-match-per-team-season is NaN) + feature stats sanity cell (.describe() + class-conditional means) + sign off 02-VALIDATION.md

### Phase 3: Baseline Model Notebooks

**Goal**: Two separate model notebooks exist that each beat the naive home-win baseline, expose a prediction function, and report per-class evaluation metrics
**Depends on**: Phase 2
**Requirements**: MODEL-01, MODEL-02, MODEL-04, EVAL-01, EVAL-02, EVAL-03
**Success Criteria** (what must be TRUE):

  1. `notebook_logistic.ipynb` runs end-to-end and reports overall accuracy above the naive ~46% home-win heuristic
  2. `notebook_random_forest.ipynb` runs end-to-end and reports macro-F1 higher than the logistic baseline
  3. Both notebooks use `TimeSeriesSplit` for cross-validation — no `KFold` or `StratifiedKFold` is present anywhere
  4. Both notebooks expose a `predict_match(home_team, away_team)` function that returns a W/D/L label using only pre-match features
  5. Both notebooks output a `classification_report` and a confusion matrix heatmap showing non-zero recall for all three classes (Home Win, Draw, Away Win)

**Plans**: TBD

### Phase 4: Advanced Model Notebook + Polish

**Goal**: A gradient boosting notebook exists that targets 65-70% accuracy and all three notebooks are submission-ready with clean markdown documentation
**Depends on**: Phase 3
**Requirements**: MODEL-03, EVAL-04
**Success Criteria** (what must be TRUE):

  1. `notebook_gradient_boost.ipynb` runs end-to-end and reports accuracy in the 65-70% range on the held-out test set (2024-2025 data)
  2. All three model notebooks contain markdown cells that explain the data source, feature engineering choices, model selection rationale, and results interpretation
  3. `predict_match(home_team, away_team)` in the gradient boost notebook returns a correct W/D/L label for a manually-verified example match

**Plans**: TBD

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Data Ingestion & Cleaning | 1/1 | Complete   | 2026-05-22 |
| 2. Feature Engineering | 1/2 | In Progress|  |
| 3. Baseline Model Notebooks | 0/0 | Not started | - |
| 4. Advanced Model Notebook + Polish | 0/0 | Not started | - |

---

*Created: 2026-05-21*
*Last updated: 2026-05-22*
