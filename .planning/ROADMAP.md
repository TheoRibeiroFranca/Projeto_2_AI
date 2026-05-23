# Roadmap: Brazilian League Match Predictor

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

**Plans:** 2/2 plans complete
Plans:
**Wave 1**

- [x] 02-01-PLAN.md — `uv sync` + define `build_features(df)` in notebook_data.ipynb (FEAT-01/02/03/04 in one function) + apply on combined train+test, re-split by season, save `dados/feature_matrix_{train,test}.parquet` (8025×31 and 1140×31, 0 NaN)

**Wave 2** *(blocked on Wave 1 completion)*

- [x] 02-02-PLAN.md — Add leakage verification cell (raw shift(1).rolling(5) reconstruction asserts first-match-per-team-season is NaN) + feature stats sanity cell (.describe() + class-conditional means) + sign off 02-VALIDATION.md

### Phase 3: Baseline Model Notebooks

**Goal**: Two separate model notebooks exist that each beat the naive home-win baseline, expose a prediction function, and report per-class evaluation metrics
**Depends on**: Phase 2
**Requirements**: MODEL-01, MODEL-02, MODEL-04, EVAL-01, EVAL-02, EVAL-03
**Success Criteria** (what must be TRUE):

  1. `notebook_logistic.ipynb` runs end-to-end and reports macro-F1 above the naive ~0.33 (primary metric; raw accuracy below 48.16% naive is expected with class_weight='balanced')
  2. `notebook_random_forest.ipynb` runs end-to-end and reports non-zero Draw recall; RF macro-F1 target is aspirational — LR (0.41) may outperform RF (0.39) with locked configs
  3. Both notebooks use `TimeSeriesSplit` for cross-validation — no `KFold` or `StratifiedKFold` is present anywhere
  4. Both notebooks expose a `predict_match(home_team, away_team)` function that returns a W/D/L label using only pre-match features
  5. Both notebooks output a `classification_report` and a confusion matrix heatmap showing non-zero recall for all three classes (Home Win, Draw, Away Win)

**Plans:** 2/2 plans complete
Plans:
**Wave 1** *(both plans are independent and can run in parallel)*

- [x] 03-01-PLAN.md — Build `notebook_logistic.ipynb`: LogisticRegression(balanced, lbfgs, max_iter=1000), TSS cross-val, classification_report, seaborn heatmap, naive baseline comparison, predict_match function
- [x] 03-02-PLAN.md — Build `notebook_random_forest.ipynb`: RandomForestClassifier(n=200, balanced, min_samples_leaf=5), TSS cross-val, classification_report, seaborn heatmap, naive baseline comparison, predict_match function

### Phase 4: Advanced Model Notebook + Polish

**Goal**: A gradient boosting notebook exists that targets 65-70% accuracy and all three notebooks are submission-ready with clean markdown documentation
**Depends on**: Phase 3
**Requirements**: MODEL-03, EVAL-04
**Success Criteria** (what must be TRUE):

  1. `notebook_gradient_boost.ipynb` runs end-to-end and reports accuracy in the 65-70% range on the held-out test set (2024-2025 data)
  2. All three model notebooks contain markdown cells that explain the data source, feature engineering choices, model selection rationale, and results interpretation
  3. `predict_match(home_team, away_team)` in the gradient boost notebook returns a correct W/D/L label for a manually-verified example match

**Plans:** 1/2 plans executed
Plans:
**Wave 1**

- [x] 04-01-PLAN.md — Build `notebook_gradient_boost.ipynb`: HistGradientBoostingClassifier(learning_rate=0.05, max_iter=300, max_depth=5, balanced), TSS cross-val, classification_report, seaborn heatmap, naive baseline comparison, predict_match function

**Wave 2** *(blocked on Wave 1 completion)*

- [ ] 04-02-PLAN.md — Add bookend markdown cells (title/intro + results interpretation) to all three model notebooks (LR, RF, GBM)

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Data Ingestion & Cleaning | 1/1 | Complete   | 2026-05-22 |
| 2. Feature Engineering | 2/2 | Complete   | 2026-05-23 |
| 3. Baseline Model Notebooks | 2/2 | Complete   | 2026-05-23 |
| 4. Advanced Model Notebook + Polish | 1/2 | In Progress|  |

---

*Created: 2026-05-21*
*Last updated: 2026-05-23*
