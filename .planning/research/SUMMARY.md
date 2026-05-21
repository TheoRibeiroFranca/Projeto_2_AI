# Research Summary: Brazilian League Match Predictor

*Synthesized: 2026-05-21*

## Executive Summary

A W/D/L classifier for Brazilian Série A matches, delivered as a Jupyter notebook. The recommended approach is **scikit-learn ensemble (RandomForestClassifier primary)** fed by rolling-window features from the 9,165-match CSV archive already in the repository. Neural networks / Keras are the wrong tool for tabular data at this scale. The ~65-70% accuracy target is the published ceiling for open-data W/D/L prediction — achievable with the table-stakes feature set.

**The single largest risk is temporal data leakage** (three independent vectors). Any one of them inflates reported accuracy 10–20 pp while producing a model that degrades to chance on real predictions. Class imbalance is the second major risk — draws (~26%) will have near-zero recall under naive settings.

---

## Stack

| Package | Pin | Action |
|---------|-----|--------|
| scikit-learn | `>=1.6,<2` | Add: `uv add scikit-learn` |
| pandas | `>=2.1,<3` | Add: `uv add pandas` |
| seaborn | `>=0.13` | Optional (confusion matrix viz) |
| requests | `>=2.32` | Already present |
| matplotlib | `>=3.8` | Already present |

**Do NOT use:** pandas 3.0 (CoW breaking changes), KFold/StratifiedKFold (temporal leakage), XGBoost unless RF stalls below 63%.

---

## Table Stakes vs Differentiators

**Table stakes — sufficient for 60–65%** (all from `campeonato-brasileiro-full.csv`):
- Home/away indicator flag
- Goals scored last 5 matches per team (rolling, shifted)
- Goals conceded last 5 matches per team
- W/D/L in each of last 5 matches per team
- Home/away win-draw-loss % for each team this season

**Free differentiators — push toward 65–70%:**
- Goal difference last 5 (derived from above)
- Points last 5 (W=3, D=1, L=0 sum)

**Defer:** Season standings, shot/possession stats (zero-filled pre-2013), formation features, player-level data, xG, betting odds.

---

## Suggested Phase Structure

**Phase 1 — Data Ingestion & Cleaning**
- Load CSVs, merge on `partida_id`, normalize team names, parse dates, audit nulls
- Pitfalls: Team name inconsistency — assert top clubs have 380+ matches after normalization
- Output: Clean `matches` DataFrame

**Phase 2 — Feature Engineering**
- Rolling 5-match windows with mandatory `.shift(1)`, home/away splits, free differentiators, NaN imputation
- Pitfalls: Random split (P1), missing shift (P2), in-game stats leakage (P4), relegation form gaps (P6), NaN row loss ~20% if `dropna` used (P13)
- Output: `feature_matrix` with verified temporal integrity; reusable `build_features()` function

**Phase 3 — Baseline Classifier & Evaluation**
- Temporal split (2003–2022 train / 2023 val / 2024–2025 test), LogisticRegression + RandomForest, `class_weight='balanced'`, `classification_report`, macro-F1 as primary metric
- Pitfalls: Draw recall collapse (P3), accuracy-only reporting (P8)
- Output: Baseline metrics; confirm beats naive ~46% home-win baseline

**Phase 4 — Advanced Model, Prediction Interface & Polish**
- GradientBoostingClassifier, `predict_match(home, away)` function, confusion matrix heatmap, optional API supplement with strict temporal controls, notebook documentation
- Pitfalls: API data leaking into training (P7), feature reconstruction divergence at inference
- Output: Final model at 65–70% accuracy target; clean notebook ready for submission

---

## Top Pitfalls

| Priority | Pitfall | Phase | Prevention |
|----------|---------|-------|------------|
| CRITICAL | Random train/test split | Phase 2 | Sort by date, split at fixed year |
| CRITICAL | Missing `.shift(1)` in rolling window | Phase 2 | Always shift before roll; verify round-1 rows are NaN |
| CRITICAL | Draw class collapse | Phase 3 | `class_weight='balanced'`; report `classification_report` |
| CRITICAL | In-game stats leakage | Phase 2 | Join stats on prior rounds only |
| CRITICAL | Team name inconsistency | Phase 1 | Build canonical name dict before any `groupby` |
| HIGH | NaN propagation drops ~20% rows | Phase 2 | Impute NaN, don't `dropna()` |
| HIGH | Accuracy-only evaluation | Phase 3 | Always include naive baseline comparison |
| MEDIUM | API data leakage into training | Phase 4 | Freeze train/test split before any API call |

---

## Key Open Questions

1. Does API-Football free tier cover Brasileirao (league ID 71)? Verify before Phase 4 — if unavailable, CSVs alone are sufficient
2. What is actual home win rate in this dataset? Verify in Phase 1 EDA (affects class weight ratios)
3. Where exactly does the stats CSV have reliable (non-zero) data? Audit in Phase 1 before including shots/possession as features
4. How many team name variants exist? Run `df['mandante'].nunique()` in Phase 1
5. Is "~65-70% accuracy" measured as overall accuracy or macro-F1? Recommend macro-F1 as primary metric

---

## Confidence

| Area | Level |
|------|-------|
| Stack (sklearn/RF/pandas) | HIGH — verified via official docs |
| Table stakes features | HIGH — all derivable from verified CSV schema |
| 65-70% accuracy ceiling | MEDIUM — widely cited; verify with actual class distribution |
| API-Football Brasileirao coverage | MEDIUM — needs verification at registration |
| Stats CSV zero-data boundary (~2013) | MEDIUM — needs Phase 1 audit |
