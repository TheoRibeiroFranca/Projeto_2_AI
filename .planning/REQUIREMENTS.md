# Requirements: Brazilian League Match Predictor

**Defined:** 2026-05-21
**Core Value:** Given any two Brazilian league teams, predict the match outcome (W/D/L) with ~65-70% accuracy using the last 5 matches of form data

## v1 Requirements

### Data

- [x] **DAT-01**: User can load and parse `campeonato-brasileiro-full.csv` as the primary historical data source (9,165 matches, 2003–2025)
- [x] **DAT-02**: System normalizes team names to a canonical mapping before any grouping or feature engineering (handles 20+ years of naming variants, e.g. "Atletico-MG" vs "Atletico MG")
- [x] **DAT-03**: System applies temporal train/test split — data sorted by date, fixed year cutoff, never random shuffle

### Features

- [ ] **FEAT-01**: System computes rolling 5-match goals scored and goals conceded per team, using `.shift(1)` before `.rolling(5)` to prevent data leakage
- [ ] **FEAT-02**: System computes W/D/L results across last 5 matches per team as a form streak
- [ ] **FEAT-03**: System computes home win/draw/loss % and away win/draw/loss % per team for the current season
- [ ] **FEAT-04**: System derives goal difference last 5 and points last 5 (W=3, D=1, L=0) from the features above (zero additional data cost)

### Models — separate notebooks

- [ ] **MODEL-01**: `notebook_logistic.ipynb` — LogisticRegression baseline with `class_weight='balanced'`, confirms model beats naive "always Home Win" heuristic (~46%)
- [ ] **MODEL-02**: `notebook_random_forest.ipynb` — RandomForestClassifier with `class_weight='balanced'` and TimeSeriesSplit cross-validation
- [ ] **MODEL-03**: `notebook_gradient_boost.ipynb` — GradientBoostingClassifier with `class_weight='balanced'` and TimeSeriesSplit cross-validation, targeting 65-70% accuracy
- [ ] **MODEL-04**: TimeSeriesSplit cross-validation (sklearn `TimeSeriesSplit`) applied inside each model notebook for hyperparameter tuning without temporal leakage

### Interface & Evaluation

- [ ] **EVAL-01**: Each notebook exposes a `predict_match(home_team, away_team)` function that returns a W/D/L label using only pre-match features
- [ ] **EVAL-02**: Each notebook outputs a `classification_report` showing precision, recall, and F1 per class (Home Win, Draw, Away Win)
- [ ] **EVAL-03**: Each notebook includes a confusion matrix heatmap
- [ ] **EVAL-04**: Each notebook has clean markdown documentation cells explaining the data, features, model choice, and results (grading considers documentation quality)

## v2 Requirements

### Data Enrichment

- **DAT-V2-01**: Supplement historical data with live/current season data from API-Football (league ID 71) for up-to-date predictions
- **DAT-V2-02**: Include shot and possession statistics from `campeonato-brasileiro-estatisticas-full.csv` (post-2013 rows only where data is non-zero)

### Advanced Features

- **FEAT-V2-01**: Head-to-head record for the two specific teams in the last 5 meetings
- **FEAT-V2-02**: Season standings position and points gap as contextual features

## Out of Scope

| Feature | Reason |
|---------|--------|
| Score / goal-range prediction | W/D/L classification is the target; scores add complexity without serving the core goal |
| Neural networks / Keras | Wrong tool for tabular data at this scale; sklearn ensembles dominate |
| Web app or API delivery | Notebook-first for v1; web layer deferred |
| Real-time betting integration | Personal/academic project, not production system |
| Leagues outside Brazilian championship | Scope is Série A only for v1 |
| Shot/possession stats (pre-2013) | Zero-filled data contaminates rolling averages; deferred to v2 |
| Player-level / formation data | Requires separate data source; adds significant complexity |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| DAT-01 | Phase 1 | Complete |
| DAT-02 | Phase 1 | Complete |
| DAT-03 | Phase 1 | Complete |
| FEAT-01 | Phase 2 | Pending |
| FEAT-02 | Phase 2 | Pending |
| FEAT-03 | Phase 2 | Pending |
| FEAT-04 | Phase 2 | Pending |
| MODEL-01 | Phase 3 | Pending |
| MODEL-02 | Phase 3 | Pending |
| MODEL-03 | Phase 4 | Pending |
| MODEL-04 | Phase 3 | Pending |
| EVAL-01 | Phase 3 | Pending |
| EVAL-02 | Phase 3 | Pending |
| EVAL-03 | Phase 3 | Pending |
| EVAL-04 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 15 total
- Mapped to phases: 15
- Unmapped: 0 ✓

---
*Requirements defined: 2026-05-21*
*Last updated: 2026-05-21 after initial definition*
