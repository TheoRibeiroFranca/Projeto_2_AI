# Domain Pitfalls: Brazilian Football Match Outcome Prediction

**Domain:** Sports match outcome classification (W/D/L)
**Data source analysed:** `dados/archive/` CSVs — 9,165 matches, 2003–2025
**Researched:** 2026-05-21

---

## Critical Pitfalls

Mistakes that cause the model to report inflated accuracy during development but fail on real prediction tasks.

---

### Pitfall 1: Temporal Data Leakage via Random Train/Test Split

**What goes wrong:** Using `train_test_split(shuffle=True)` or any random split on a time-ordered dataset. A match from round 30 of 2019 ends up in training while a match from round 5 of 2019 ends up in test. The rolling-window features for the test match (e.g., "goals scored in last 5 games") are computed from future matches that the model will have already seen during training.

**Why it happens:** Sklearn's `train_test_split` defaults to `shuffle=True`. This is correct for i.i.d. data (images, text) and wrong for time series. It is the single most common error in sports prediction notebooks.

**Consequences:** Accuracy looks 5–15 percentage points higher than it truly is. The model appears to beat the 65–70% target but degrades to ~50% on truly unseen future matches.

**Prevention:**
- Always split by time: train on seasons up to year N, test on year N+1.
- With the Brazilian championship data (2003–2025), a clean split is: train on 2003–2022, validate on 2023, test on 2024–2025.
- Use `df.sort_values('data')` before any split; never pass `shuffle=True` to any splitter.
- Wrap feature engineering in a function that takes a cutoff date and only looks backward from that date.

**Warning signs:**
- `train_test_split` appears anywhere in the notebook without `shuffle=False`.
- Feature computation uses the full DataFrame before splitting.
- Validation accuracy jumps more than expected when adding rolling features.

**Phase to address:** Feature engineering phase — enforce temporal split before writing a single feature.

---

### Pitfall 2: Future-Aware Rolling Feature Computation (Lookahead)

**What goes wrong:** Computing "last 5 goals scored" for a match using a `groupby().rolling()` or `.shift()` applied to the full dataset. If the shift is off by one, the feature for match N includes the result of match N itself. This is lookahead leakage — the label is embedded in the feature.

**Why it happens:** `pandas.DataFrame.rolling()` is inclusive of the current row by default. `shift(1)` is correct; `shift(0)` or omitting the shift leaks the current match result.

**Data-specific risk:** The `campeonato-brasileiro-full.csv` contains the final score (`mandante_Placar`, `visitante_Placar`) alongside each row. Any feature that aggregates these columns without excluding the current row leaks the exact result being predicted.

**Consequences:** Model achieves near-perfect accuracy during evaluation but predicts randomly on future matches where the result is unknown.

**Prevention:**
- Always use `shift(1)` before any `rolling()` aggregation.
- Formula: `df.groupby('team')['goals_scored'].shift(1).rolling(5).mean()` — shift first, then roll.
- After feature engineering, inspect the feature for the first match of each team's season: it must be NaN (no prior data), not a valid number.
- Never include `mandante_Placar` or `visitante_Placar` as raw features — only as ingredients in shifted aggregations.

**Warning signs:**
- Feature for round 1 of a season is non-null for any team.
- `.rolling()` applied before `.shift()`.
- Match result columns appear directly as model inputs.

**Phase to address:** Feature engineering phase, enforced via a unit test on round-1 rows.

---

### Pitfall 3: Class Imbalance Ignored for the Draw Class

**What goes wrong:** In football, draws occur roughly 25–28% of the time. Home wins ~45%, away wins ~28%. A model trained with standard cross-entropy and no class weighting learns to almost never predict "Draw" because predicting Home Win or Away Win is nearly always safer from a loss perspective. The model reaches 50–55% accuracy by mostly predicting Home Win, and the Draw class recall collapses to near zero.

**Data-specific check:** The Brazilian championship data from 2003–2025 shows consistent home advantage; the expected class frequencies are roughly Home Win 45%, Draw 26%, Away Win 29%. Verify this before training.

**Consequences:** The 65–70% accuracy target is achievable by almost never predicting draws — but the model is practically useless because it cannot distinguish the most ambiguous match type.

**Prevention:**
- Compute class weights: `sklearn.utils.class_weight.compute_class_weight('balanced', ...)`.
- Pass `class_weight` to `model.fit()` (Keras) or `sample_weight` to sklearn estimators.
- Report per-class F1, precision, recall (via `classification_report`) alongside overall accuracy.
- Consider using weighted accuracy or macro-F1 as the primary evaluation metric.

**Warning signs:**
- Draw recall in `classification_report` below 0.20.
- Model predicts "Draw" for fewer than 15% of test examples despite a 25–28% true prevalence.
- Only overall accuracy reported, no per-class breakdown.

**Phase to address:** Model training phase; evaluation phase must include `classification_report`.

---

### Pitfall 4: In-Game Statistics Used as Pre-Match Features

**What goes wrong:** The `campeonato-brasileiro-estatisticas-full.csv` contains in-game statistics — shots, possession, passes, fouls, cards — recorded per match. These are recorded after/during the match. Using them as input features for predicting match outcome is direct label leakage: possession % and shots on target causally determine the result.

**Why it is subtle:** These columns look like "team quality" features. A builder might reason "good teams have high possession on average" and compute a rolling average of these stats. That is correct if done with a `shift(1)` — using only stats from prior matches. It is wrong if any current-match stat row is included.

**Data-specific risk:** The statistics CSV has ~18,000 rows (2 per match). The early years (2003 through approximately 2013) show all-zero values for most columns, meaning these features will have zero variance for those seasons and produce silent bugs rather than obvious errors.

**Prevention:**
- Treat in-game stats as post-match data. When joining stats to match rows for feature engineering, always join on prior rounds only.
- For the zero-data period (pre-2014 approximately), explicitly mark these features as missing rather than zero.
- Consider restricting the training dataset to the period where statistics data is reliable (round 1 of ~2014 onward) rather than imputing zeros.

**Warning signs:**
- `campeonato-brasileiro-estatisticas-full.csv` joined on `partida_id` without any temporal shift.
- Rolling averages of possession/shots that are non-null for round 1 rows.
- Unusually high feature importance on possession or shots columns.

**Phase to address:** Data loading and feature engineering phases.

---

### Pitfall 5: Team Name Inconsistency Across Seasons

**What goes wrong:** The same club appears under different names across the 2003–2025 dataset. For example, "Atletico-MG" vs "Atletico MG" vs "Atlético Mineiro", or "Botafogo-RJ" vs "Botafogo". Rolling aggregations computed via `groupby('mandante')` silently treat these as separate teams, producing wrong or missing form statistics.

**Data-specific evidence:** Even in the first 20 rows of the CSV the naming is not fully consistent, and the dataset spans multiple data-entry eras (2003 vs. 2025). Promoted/relegated clubs add additional edge cases.

**Consequences:** Teams with name variants appear to have shorter histories than they actually do; their rolling features are computed from fewer games, biasing form estimates.

**Prevention:**
- Build a canonical team name mapping before any feature engineering.
- Normalise names with lowercasing, accent stripping, and explicit alias mapping before groupby.
- Assert that a known set of top teams (Flamengo, Palmeiras, Corinthians, etc.) each have at least 380 matches (38 rounds × 10+ seasons) in the training set after normalisation.

**Warning signs:**
- `df['mandante'].nunique()` returns more than ~45 distinct teams (more than the ~20/season with promotion/relegation variation).
- A top-tier club has fewer than 200 historical matches in the grouped data.

**Phase to address:** Data loading phase, before any feature engineering.

---

### Pitfall 6: Promotion/Relegation Breaks Team Form Continuity

**What goes wrong:** A team that was relegated after 2018 and promoted back in 2021 has a three-year gap in Série A records. If rolling features look back 5 matches without date-gap awareness, they will either produce NaN (breaking model input) or pull in stale pre-relegation data from 2018 as if it were recent form.

**Why it is specific to Brazilian football:** The Brasileirao has 4 relegations and 4 promotions per season. Over a 22-season dataset, almost every mid-table club has at least one relegation. The problem is especially acute for clubs like Cruzeiro (relegated 2019, promoted 2023) or Vasco.

**Prevention:**
- When computing rolling form, filter each team's match history to only matches that fall within 365 days of the prediction date, not simply the last N rows.
- Alternatively, treat each team's Série A spell as a separate entity for form computation.
- Any NaN rolling features for round 1 re-entrants should be imputed with league-wide averages, not zeros.

**Warning signs:**
- Form features are non-null for a newly promoted team's first match of their return season.
- A team with a known relegation gap has identical feature values on either side of the gap.

**Phase to address:** Feature engineering phase.

---

### Pitfall 7: Overfitting on Small Recent Data via API Supplement

**What goes wrong:** The plan is to supplement historical CSVs with live/current data from an external football API. If the API data is used to create a second fine-tuning phase (similar to the MNIST notebook pattern already in the repo), the model can overfit the distribution of the current season — which is exactly the test set. This is analogous to the Phase 2 overfitting described in CONCERNS.md (`val_accuracy` hitting 1.0 on 320 samples).

**Why it is tempting:** Recent form is the most predictive feature. Adding the current season to training feels like improving recency. But if the test set is drawn from the same current season, any overlap creates leakage.

**Prevention:**
- Keep the most recent season as a held-out test set. Never train on any portion of it.
- If the API provides the current ongoing season, use it only for generating features for prediction, not as training labels.
- Define the train/val/test cut before touching the API data.

**Warning signs:**
- API data is fetched inside the training loop or before the train/test split.
- Test set contains matches from the same season as the API data.

**Phase to address:** Data loading phase (enforce the split before API calls).

---

### Pitfall 8: Accuracy as the Only Metric

**What goes wrong:** Reporting only overall accuracy obscures the model's actual behaviour. A model that always predicts "Home Win" achieves ~45% accuracy on Brazilian championship data — far below the 65–70% target. But a model at 62% overall accuracy that has Draw recall of zero is also nearly useless despite looking better.

**Why it is common:** Keras's default metric for multi-class classification is accuracy. The notebook will show accuracy in training history plots and that becomes the primary reported number.

**Prevention:**
- Always include `sklearn.metrics.classification_report` in the evaluation cell.
- Report macro-averaged F1 alongside accuracy.
- Establish a naive baseline (most-frequent-class classifier) before reporting model results — the improvement over baseline is the real metric.
- For the 65–70% target, clarify whether this is overall accuracy, weighted F1, or something else.

**Warning signs:**
- Only `model.evaluate()` appears in evaluation cells.
- No `classification_report` or confusion matrix.
- No comparison to a "always predict Home Win" or "predict by historical home/away ratio" baseline.

**Phase to address:** Evaluation phase (but define metrics before training so training is not over-optimised for the wrong signal).

---

### Pitfall 9: Using Match-Day Features Not Available Before Kickoff

**What goes wrong:** Some features that seem like "context" are actually only known after the match begins or ends. Attendance (`arrecadacao` in the CSV) is typically reported post-match. Formation (`formacao_mandante`, `formacao_visitante`) may not be publicly available until the team sheet is announced 1 hour before kickoff, not at the time of prediction. Including these as training features is fine for historical modelling but creates an inconsistency when the model is used for upcoming match prediction.

**Data-specific note:** The CSV has `formacao_mandante` and `formacao_visitante` populated for recent seasons. Using formation history as a rolling feature is safe; using current-match formation as a direct input is only safe if the intended use is post-announcement prediction.

**Prevention:**
- Separate features into "always available before kickoff" (team identity, historical form, home/away record, season standings) vs. "conditionally available" (formation, team sheet).
- Document which feature class each input belongs to.
- For v1, restrict to always-available features to keep the prediction interface simple.

**Warning signs:**
- `arrecadacao` (revenue/attendance) appears as a direct model feature.
- Current-match formation used as a direct input (not rolling average of past formations).

**Phase to address:** Feature engineering phase design (categorise features before building them).

---

## Moderate Pitfalls

---

### Pitfall 10: Symmetric Feature Construction Breaks Home/Away Distinction

**What goes wrong:** A model built with features like "team A goals last 5" and "team B goals last 5" — symmetrically — loses the home/away signal. Home advantage is one of the strongest predictors in football (~5–7% accuracy lift). If the features do not encode which team is home and which is away, the model is forced to learn this from team identity alone, which is noisier and less generalisable.

**Prevention:**
- Always construct separate features for "home team form" and "away team form" — never abstract them as "team A" and "team B".
- Include an explicit "is home" flag even though it is implicit in feature ordering.
- Compute rolling stats separately for home games vs. all games (home-specific form vs. overall form).

**Phase to address:** Feature engineering phase.

---

### Pitfall 11: Season Reset Not Handled in Rolling Windows

**What goes wrong:** A rolling 5-match window at round 1 of a new season picks up matches from the end of the previous season. For most teams this is fine. But for promoted teams it is wrong (they were in Série B), and for all teams the pre-season break means those "last 5" matches are 3–4 months stale.

**Prevention:**
- Constrain rolling windows to the current season only, or to matches within the last 90 days.
- At round 1, all form features should be NaN or imputed (not pulled from prior season).

**Phase to address:** Feature engineering phase.

---

### Pitfall 12: Neural Network Chosen by Default Instead of Gradient Boosting

**What goes wrong:** The existing repo uses Keras/TensorFlow (MNIST project). It is tempting to use the same stack for the football predictor. Tabular data with ~9,000 rows and ~20–30 engineered features is a regime where gradient-boosted trees (XGBoost, LightGBM) reliably outperform or match neural networks and require far less hyperparameter tuning.

**Prevention:**
- Establish a LightGBM or XGBoost baseline before investing in neural network architecture.
- Use neural networks only if they demonstrably outperform tree models on the validation set.
- The Keras stack can still be used if the project requires it, but set expectations accordingly.

**Phase to address:** Model selection phase.

---

## Minor Pitfalls

---

### Pitfall 13: NaN Propagation in Rolling Features Causes Silent Data Loss

**What goes wrong:** Rolling windows over the first few matches of each team produce NaN. If NaN rows are dropped with `dropna()`, the round 1–5 data is silently removed. For 20 teams per season × 22 seasons, this can silently remove 2,000+ rows from training — roughly 20% of the dataset.

**Prevention:**
- Impute NaN rolling features (with league average or zero, documented) rather than dropping rows.
- Assert `len(df_after_dropna) == len(df_before_dropna)` or explicitly log dropped row counts.

**Phase to address:** Feature engineering phase.

---

### Pitfall 14: Draw Prediction Collapses Under Wrong Loss Function

**What goes wrong:** Standard categorical cross-entropy with equal class weights causes the model to avoid predicting draws because the penalty for a wrong draw prediction is the same as for a wrong home-win prediction, but home wins are more frequent and thus easier to get right on average. The model minimises loss by biasing toward majority classes.

**Prevention:**
- Use class-weighted cross-entropy from the start of training.
- Log the predicted class distribution on validation after each epoch; flag if draw predictions drop below 10%.

**Phase to address:** Model training phase.

---

### Pitfall 15: Test Set Contamination via Hyperparameter Tuning

**What goes wrong:** The test set is used multiple times — to check performance, to choose hyperparameters, to decide when to stop adding features — and begins to function as a validation set. The reported test accuracy becomes optimistic.

**Prevention:**
- Define train/validation/test split once, early.
- Touch the test set exactly once: final evaluation after all development decisions are frozen.
- Use the validation set for all model development decisions.

**Phase to address:** Evaluation phase setup (must be defined before any modelling begins).

---

## Phase-Specific Warning Map

| Phase | Most Likely Pitfall | Mitigation |
|-------|--------------------|-----------| 
| Data loading | Team name inconsistency (P5), statistics zero-data era (P4) | Normalise names immediately; document data quality gaps by year |
| Feature engineering | Temporal leakage via random split (P1), lookahead in rolling features (P2), future-aware stats (P4), season/relegation gaps (P6) | Build a `compute_features(df, cutoff_date)` function that enforces all temporal constraints |
| Model training | Class imbalance / draw collapse (P3, P14), wrong architecture choice (P12) | Compute class weights before `model.fit`; benchmark LightGBM first |
| Evaluation | Accuracy-only reporting (P8), test set contamination (P15) | Require `classification_report` and naive baseline comparison in every evaluation cell |
| API integration | Leakage from current-season data into training (P7), pre-kickoff feature availability (P9) | Freeze train/test split before any API call; document feature availability category |

---

## Sources

- Data characteristics inferred from direct analysis of `dados/archive/campeonato-brasileiro-full.csv` and `campeonato-brasileiro-estatisticas-full.csv` (9,165 matches, 2003–2025).
- Pitfalls P1–P3 and P8 are well-documented in sports analytics literature and multiple Kaggle competition post-mortems for football prediction tasks (temporal leakage is cited as the #1 error in virtually all sports ML retrospectives).
- Pitfalls P5, P6, P11 are specific to long-horizon Brazilian championship data; verified by inspection of the CSV date range and the known relegation history of clubs like Cruzeiro and Vasco.
- Pitfall P4 (in-game stats as features) verified by examining the `campeonato-brasileiro-estatisticas-full.csv` column semantics against the `Legenda.txt` field descriptions.
- Pitfall P7 noted due to structural similarity to the existing MNIST notebook's two-phase fine-tuning pattern documented in `.planning/codebase/CONCERNS.md`.
