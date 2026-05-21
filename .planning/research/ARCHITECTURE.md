# Architecture Patterns: Brazilian Football Match Outcome Predictor

**Domain:** Sports match outcome classification (W/D/L)
**Data profile:** ~9,165 historical matches (2003–present) from local CSVs, live API supplement
**Delivery format:** Jupyter notebook, single-file
**Researched:** 2026-05-21

---

## Recommended Architecture

A linear notebook pipeline with five clearly bounded logical layers. Each layer has a single responsibility and passes data forward in a defined shape. No layer reaches back upstream.

```
┌──────────────────────────────────────────────────────────────┐
│                  football_predictor.ipynb                    │
│                (single-file deliverable)                     │
└──────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 1 — Data Ingestion                                    │
│  Sources: dados/archive/ CSVs + live API                     │
│  Output: raw_matches DataFrame, raw_stats DataFrame          │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 2 — Data Merging & Cleaning                           │
│  Join CSVs on partida_id, resolve team name variants,        │
│  parse dates, sort chronologically                           │
│  Output: matches DataFrame (one row per match, clean)        │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 3 — Feature Engineering                               │
│  Rolling 5-match windows per team (home/away/overall)        │
│  Output: feature_matrix DataFrame (one row per match,        │
│          all features numeric, target label encoded)         │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 4 — Model Training & Evaluation                       │
│  Temporal train/test split → sklearn/Keras classifier        │
│  Output: trained model, evaluation metrics                   │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 5 — Prediction Interface & Visualization              │
│  predict_match(home_team, away_team) function                │
│  Confusion matrix, accuracy curves, example predictions      │
└──────────────────────────────────────────────────────────────┘
```

---

## Component Boundaries

| Component | Responsibility | Inputs | Outputs | Must NOT do |
|-----------|---------------|--------|---------|-------------|
| Data Ingestion | Load all raw data into DataFrames | CSV files, API responses | `raw_matches`, `raw_stats` | Compute features, filter by season |
| Merging & Cleaning | Produce a single clean match table | `raw_matches`, `raw_stats` | `matches` (1 row/match) | Compute rolling windows |
| Feature Engineering | Compute rolling-window features per team | `matches` | `feature_matrix` | Touch the model |
| Model Training | Fit classifier, evaluate on held-out set | `feature_matrix` | `model`, metrics | Re-derive features |
| Prediction Interface | Expose human-readable prediction API | `model`, current form data | Probability vector + label | Retrain model |

Keeping these boundaries strict prevents the most common notebook anti-pattern: feature logic scattered across training cells that cannot be reproduced at inference time.

---

## Data Flow

### Training Path

```
dados/archive/campeonato-brasileiro-full.csv
        ── partida_id, mandante, visitante,
           mandante_Placar, visitante_Placar,
           vencedor, rodata, data ──────────────────────────┐
                                                            │
dados/archive/campeonato-brasileiro-estatisticas-full.csv   │
        ── partida_id, clube, chutes, chutes_no_alvo,        │
           posse_de_bola, passes, faltas, escanteios ────────┤
                                                            │
Live API (optional supplement for recent seasons)           │
        ── same schema, newer matches ────────────────────── ┘
                                                            │
                           MERGE on partida_id              │
                           WIDE format: one row per match   │
                           with home & away stats side by   │
                           side                             │
                                                            ▼
                    matches DataFrame
                    ┌──────────────────────────────────────┐
                    │ match_id, date, round,                │
                    │ home_team, away_team,                 │
                    │ home_goals, away_goals,               │
                    │ home_shots, away_shots,               │
                    │ home_possession, away_possession, ... │
                    │ label: {H, D, A}                      │
                    └──────────────────────────────────────┘
                                                            │
            SORT by date, GROUP by team                     │
            ROLLING 5-match window (shift(1) to avoid       │
            leakage — never include the match being          │
            predicted in its own window)                    │
                                                            ▼
                    feature_matrix DataFrame
                    ┌──────────────────────────────────────┐
                    │ home_goals_scored_last5               │
                    │ home_goals_conceded_last5             │
                    │ home_wins_last5, home_draws_last5,    │
                    │ home_losses_last5                     │
                    │ away_goals_scored_last5               │
                    │ away_goals_conceded_last5             │
                    │ away_wins_last5, away_draws_last5,    │
                    │ away_losses_last5                     │
                    │ home_home_record_last5  (home/away    │
                    │ away_away_record_last5   venue-split) │
                    │ ... (shots, possession, corners, etc) │
                    │ label: {0=H, 1=D, 2=A}               │
                    └──────────────────────────────────────┘
                                                            │
            TEMPORAL SPLIT (not random):                    │
            train = seasons before cutoff year              │
            test  = final season(s)                         │
                                                            ▼
                    model.fit(X_train, y_train)
                    model.evaluate(X_test, y_test)
```

### Inference Path (Prediction Interface)

```
predict_match(home_team="Flamengo", away_team="Palmeiras")
        │
        ▼
Fetch last-5 matches for each team from feature_matrix
        (or re-query merged DataFrame if feature_matrix
         is rebuilt fresh)
        │
        ▼
Assemble single feature vector [1 x n_features]
        │
        ▼
model.predict_proba(feature_vector)
        → [P(H), P(D), P(A)]
        │
        ▼
Return: predicted_label + probabilities
```

---

## Critical Data Flow Rule: Temporal Integrity

**The rolling window must use `.shift(1)` before computing.**

For any match at index i, the window must include only matches at indices < i. Using the current match's own statistics to compute its features leaks the result into the training data — this is the single most common error in sports prediction notebooks and inflates test accuracy by 10–20 percentage points, producing a model that cannot predict anything in production.

Correct pattern:
```python
team_stats = matches_per_team.sort_values('date')
team_stats['goals_last5'] = (
    team_stats['goals']
    .shift(1)           # exclude current match
    .rolling(5, min_periods=1)
    .mean()
)
```

---

## Suggested Build Order

Dependencies between layers determine the correct build sequence.

```
1. Data Ingestion
   Reason: Everything downstream depends on having raw DataFrames.
   Risk: CSV encoding issues (latin-1 / utf-8), team name inconsistencies
         across seasons, sparse statistics columns in early seasons.

2. Merging & Cleaning
   Reason: Feature engineering cannot run until matches have a clean
           unified schema. Team name normalization must happen here —
           "Athletico-PR" vs "Atletico PR" vs "CAP" appear in the CSVs.
   Risk: Wide merge on partida_id produces NaN for stat columns in
         matches missing from estatisticas CSV (early seasons).

3. Exploratory Data Analysis (inline, minimal)
   Reason: Verify label distribution (H/D/A class imbalance),
           check how many matches have NaN features, validate
           rolling window output visually on 2–3 known teams.
   Risk: Skipping this step leads to training on silently wrong features.

4. Feature Engineering
   Reason: Must be built and validated before any model sees data.
           All features must be computable from data known before
           the match (no same-day statistics).
   Risk: Leakage via missing .shift(1); venue-split records require
         separate home/away sub-DataFrames.

5. Baseline Classifier (sklearn: LogisticRegression or RandomForest)
   Reason: Establish a fast feedback loop on feature quality before
           investing in Keras. If baseline does not beat the naive
           "always predict Home Win" (~46% in Brazilian league),
           the features are wrong — fix features before adding model
           complexity.
   Risk: Skipping baseline means debug effort goes into model when
         the real problem is feature leakage or normalization.

6. Neural Network / Advanced Classifier (Keras MLP or XGBoost)
   Reason: Only worthwhile once baseline confirms features are valid.
   Risk: Class imbalance (Draws ~26% vs Home Win ~46%) needs
         class_weight or oversampling; Keras default ignores this.

7. Prediction Interface & Visualization
   Reason: Final polish; safe to build only after model is validated.
   Risk: Feature reconstruction at inference must be identical to
         training — any divergence silently degrades predictions.
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Data Leakage via Non-Shifted Rolling Window

**What:** Computing `goals_last5` using the current match's own goals before shifting.
**Why bad:** Inflates test accuracy 10–20 pp; model is useless for future matches.
**Instead:** Always `.shift(1)` before `.rolling().mean()` or `.rolling().sum()`.

### Anti-Pattern 2: Random Train/Test Split

**What:** Using `sklearn.model_selection.train_test_split` with `shuffle=True` (default).
**Why bad:** Future matches leak into training set; metrics are optimistic and unreliable.
**Instead:** Sort by date, split at a fixed date boundary:
```python
cutoff = '2022-01-01'
X_train = feature_matrix[feature_matrix['date'] < cutoff]
X_test  = feature_matrix[feature_matrix['date'] >= cutoff]
```

### Anti-Pattern 3: Team Name Inconsistency Not Resolved

**What:** "Atletico-MG", "Atletico MG", "Galo" treated as different teams.
**Why bad:** Rolling window computes form for a phantom team; most rows get NaN features.
**Instead:** Build a canonical team name mapping dict and apply it at the Merging layer before any groupby.

### Anti-Pattern 4: API Data Mixed into Feature Matrix Without Temporal Alignment

**What:** Appending API match rows before sorting by date; rows land in wrong chronological position.
**Why bad:** Rolling windows compute incorrect form sequences.
**Instead:** Concatenate raw_matches DataFrames (CSV + API), then sort once globally by date before any feature engineering.

### Anti-Pattern 5: Feature Reconstruction Divergence at Inference

**What:** Training uses a precomputed DataFrame; prediction cell re-derives features with a slightly different formula.
**Why bad:** Silent accuracy degradation; no error is raised.
**Instead:** Write a single `build_features(matches_df)` function called by both the training cell and the prediction interface.

---

## Scalability Considerations

This is a ~9K row dataset — scalability is not a concern. The relevant concern is correctness and reproducibility.

| Concern | At current scale (9K rows) | If extended to live updates |
|---------|---------------------------|----------------------------|
| Feature engineering | Pandas in-memory, fine | Same; re-run on append |
| Model retraining | Seconds (sklearn), minutes (Keras) | Acceptable for weekly refresh |
| API quota | Not applicable | Cache API responses locally to avoid re-fetching |
| Team name changes | Manageable with a dict | Needs ongoing maintenance as leagues add/drop clubs |

---

## Component Communication Summary

```
Layer 1 ──── raw DataFrames ────► Layer 2
Layer 2 ──── clean matches ──────► Layer 3
Layer 3 ──── feature_matrix ─────► Layer 4
Layer 4 ──── trained model ──────► Layer 5
Layer 3 ──── (same features) ───► Layer 5 (inference reconstruction)
```

The only non-linear dependency is Layer 3 → Layer 5: the inference path in Layer 5 must reuse the same feature logic as Layer 3. This is the one coupling point that needs explicit design attention — a shared `build_features()` function or clearly duplicated but identical logic.

---

## Build Order Implications for Roadmap

Phases map naturally to layers:

- **Phase 1** should cover Layers 1–2 (data loading, merging, cleaning). Deliverable: clean `matches` DataFrame with verified row count and no unnamed duplicates.
- **Phase 2** should cover Layer 3 (feature engineering). Deliverable: `feature_matrix` with verified temporal integrity (manual spot-check on 2 teams).
- **Phase 3** should cover Layer 4 baseline (sklearn classifier). Deliverable: accuracy above naive baseline (~46%) on held-out test.
- **Phase 4** should cover Layer 4 advanced model (Keras or XGBoost) + Layer 5 prediction interface + visualizations. Deliverable: notebook meeting 65–70% accuracy target with clean documentation.

**Phases 1–2 cannot be parallelized** — cleaning must complete before feature engineering starts.
**Phases 3–4 cannot be parallelized** — baseline must validate features before advanced model investment.

---

*Architecture analysis: 2026-05-21*
*Confidence: HIGH for pipeline structure (well-established ML pattern); MEDIUM for specific feature enumeration (depends on actual data quality found during Phase 1); HIGH for anti-patterns (leakage and temporal split errors are universally documented in sports prediction literature).*
