# Technology Stack

**Project:** Brazilian Football Match Outcome Predictor
**Researched:** 2026-05-21
**Research confidence:** MEDIUM-HIGH (core ML stack HIGH via official docs; API options MEDIUM via training knowledge + partial web verification)

---

## Recommended Stack

### Core ML — Classifier

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| scikit-learn | 1.6.1 (current stable; 1.8.0 released Dec 2025 — upgrade if UV allows) | All classification, preprocessing, evaluation | The de-facto standard for tabular ML; RandomForest and GradientBoosting are the go-to classifiers for structured, low-sample-count problems like this one |
| RandomForestClassifier | (scikit-learn) | Primary classifier | Outperforms HistGradientBoosting on small datasets (<10k rows) because histogram binning creates overly approximate splits at low sample counts; confirmed by sklearn official docs |
| GradientBoostingClassifier | (scikit-learn) | Secondary / ensemble member | Better than HistGB on small datasets; use as comparison baseline alongside RF |
| LogisticRegression | (scikit-learn) | Baseline and probability calibration | Required as a naive baseline to verify the RF/GB models are actually learning; also interpretable |

**Why NOT XGBoost/LightGBM:** Both are excellent gradient boosting libraries, but they add a dependency not already in the project, require separate installation, and provide marginal gains over sklearn's `GradientBoostingClassifier` on datasets of this size (~1,000–5,000 usable rows after feature engineering). The existing TF/Keras env does not benefit from them. Stick with sklearn unless the RF/GB ceiling is hit and accuracy is meaningfully below 65%.

**Why NOT Keras neural networks for this task:** Deep learning requires large datasets to outperform gradient boosting on tabular data. With ~5,000 historical matches and ~15 engineered features, a neural network will overfit unless carefully regularized, and still typically loses to RF/GB. The existing Keras stack is irrelevant for this deliverable — do not port the MNIST approach to this problem.

---

### Data Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pandas | 2.3.x (latest stable Jun 2025; 3.0.0 exists but avoid — breaking changes to string dtype and CoW) | CSV loading, rolling aggregations, feature engineering | Already in the existing env via TF dependencies; essential for time-series windowing (last-5-matches rolling stats) |
| numpy | 2.0.2 (locked in uv.lock) | Array operations | Already present; no additional installation needed |

**Why NOT pandas 3.0:** Pandas 3.0 (Jan 2026) introduces Copy-on-Write as default and dedicated string dtype. This breaks common patterns like `df['col'] = ...` chained indexing. Stick with 2.3.x until the project stabilizes.

---

### Live Data API

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| API-Football (api-sports.io) | v3 | Supplement CSV history with current season data | Covers Brazilian Serie A (Brasileirao); free tier provides 100 requests/day which is sufficient for batch feature refresh; returns structured JSON with fixtures, results, team statistics |
| requests | 2.33.1 (already locked) | HTTP client for API calls | Already in `pyproject.toml`; no new dependency needed |

**API-Football free tier notes (MEDIUM confidence — training knowledge, partially verified):**
- 100 requests/day on free plan
- Covers Brasileirao Serie A (league ID 71)
- Returns home/away goals, result, match date — exactly the fields needed
- Requires registration at api-sports.io for an API key

**Alternative if API-Football proves unreliable:** football-data.org also covers South American leagues on paid tiers (the free tier is European-only). For a purely academic project, scraping the existing CSV data to its current year and treating the API as optional is a valid fallback. Do not block feature engineering on API availability.

**Why NOT a web scraper:** Scraping (e.g., `beautifulsoup4` against Wikipedia or cbf.com.br) is fragile, legally gray, and slower to implement. A structured API is preferable. If the free API proves insufficient, fall back to extending the existing CSV data manually.

---

### Feature Engineering

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pandas rolling/groupby | (pandas) | Last-5-match rolling windows per team | Standard pattern; `df.groupby('team').rolling(5)` computes goals scored/conceded/streak without extra libraries |
| scikit-learn Pipeline | (scikit-learn) | Chain StandardScaler → classifier | Prevents data leakage by fitting scaler only on training fold; mandatory for correct cross-validation |
| scikit-learn StandardScaler | (scikit-learn) | Normalize continuous features | Required for LogisticRegression baseline; RF/GB are scale-invariant but it's harmless to include and keeps the pipeline consistent |

---

### Validation Strategy

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| scikit-learn TimeSeriesSplit | (scikit-learn) | Cross-validation respecting temporal order | CRITICAL: Standard KFold causes data leakage by training on future matches and testing on past ones. TimeSeriesSplit grows the training window forward, preserving causal order. |
| scikit-learn classification_report | (scikit-learn) | Per-class precision, recall, F1 | Draws / rare outcomes will have lower recall; `classification_report` exposes class-level performance which accuracy alone hides |
| scikit-learn confusion_matrix | (scikit-learn) | Error analysis | Home win / Draw / Away win confusion is the key diagnostic |
| scikit-learn balanced_accuracy_score | (scikit-learn) | Primary reported metric | W/D/L classes are imbalanced (home wins ~45%, draws ~25%, away wins ~30% in Brazilian football); balanced accuracy accounts for this |

---

### Visualization

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| matplotlib | >=3.8 (already in pyproject.toml) | Feature distribution plots, confusion matrix heatmap, training curves | Already present; no new dependency |
| seaborn | >=0.13 | Styled confusion matrix heatmap | Thin wrapper over matplotlib; `seaborn.heatmap` produces publication-quality confusion matrices in ~3 lines. Requires `pip install seaborn` — lightweight, no conflicts |

**Why seaborn:** Pure matplotlib confusion matrices require manual annotation. `seaborn.heatmap(cm, annot=True)` is the standard Jupyter notebook pattern. The alternative is `ConfusionMatrixDisplay` from sklearn, which is also acceptable and requires zero new dependencies — prefer that if adding seaborn feels heavy.

---

### Notebook Environment

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Jupyter Notebook / JupyterLab | (existing, ipykernel 7.2.0) | Delivery format | Already installed; the new notebook is a separate file from notebook.ipynb (MNIST). Name it `football_predictor.ipynb` |
| UV | (existing) | Dependency management | Add new packages via `uv add pandas scikit-learn seaborn`; do not install via raw pip to preserve lockfile integrity |

---

## What NOT to Use

| Category | Avoid | Reason |
|----------|-------|--------|
| Classifier | Keras/TensorFlow neural network | Wrong tool for small tabular dataset; overfits without large data; adds complexity without accuracy gains |
| Classifier | XGBoost, LightGBM | Extra dependencies, marginal gains at this dataset size; add only if RF/GB ceiling is below 63% |
| Classifier | SVM (SVC) | Slow on this feature set without tuning; no probability output by default; lower ceiling than ensemble methods here |
| Validation | KFold, StratifiedKFold | Causes temporal data leakage; do not use for time-ordered match data |
| Validation | Train/test split only | Single split is high-variance at this dataset size; TimeSeriesSplit is required |
| Data | pandas 3.0.0 | Breaking CoW and string dtype changes; avoid until stable patterns established |
| API | Web scraping (BeautifulSoup, Selenium) | Fragile, legally ambiguous, slower than structured API |
| Visualization | Plotly, Bokeh | Interactive plots add complexity in a graded notebook context; matplotlib/seaborn is the expected format |

---

## Installation Delta (What to Add)

The existing `pyproject.toml` is missing: `scikit-learn`, `pandas`, and optionally `seaborn`.

```bash
# Add via UV (preserves lockfile integrity)
uv add scikit-learn pandas seaborn
```

Note: `requests` (already present at 2.33.1) covers the API HTTP calls. No additional HTTP library needed.

---

## Dependency Versions (Summary)

| Package | Recommended Pin | Status |
|---------|----------------|--------|
| scikit-learn | `>=1.6,<2` | Not in pyproject.toml — add |
| pandas | `>=2.1,<3` | Not in pyproject.toml — add |
| seaborn | `>=0.13` | Not in pyproject.toml — optional add |
| requests | `>=2.32` | Already present |
| matplotlib | `>=3.8` | Already present |
| numpy | `>=1.26,<2.2` | Already present |
| tensorflow | `==2.18.*` | Already present (not used for football model) |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| scikit-learn version (1.6.1 / 1.8.0) | HIGH | Verified via official scikit-learn docs |
| RandomForest preferred over HistGB at small scale | HIGH | Confirmed via sklearn official docs (explicit recommendation) |
| pandas 2.3.x version | HIGH | Verified via pandas release notes |
| TimeSeriesSplit necessity | HIGH | Confirmed via sklearn cross-validation docs |
| API-Football covering Brasileirao | MEDIUM | Training knowledge; could not verify via WebFetch due to tool restrictions |
| API-Football free tier limits (100 req/day) | MEDIUM | Training knowledge; requires verification at api-sports.io before implementation |
| XGBoost/LightGBM version numbers | LOW | Not directly verified; recommend deferring until needed |

---

## Sources

- scikit-learn 1.8.0 release notes: https://scikit-learn.org/stable/whats_new.html (fetched 2026-05-21)
- scikit-learn ensemble docs (RF vs HistGB small dataset recommendation): https://scikit-learn.org/stable/modules/ensemble.html (fetched 2026-05-21)
- scikit-learn TimeSeriesSplit docs: https://scikit-learn.org/stable/modules/cross_validation.html#time-series-split (fetched 2026-05-21)
- scikit-learn classification metrics: https://scikit-learn.org/stable/modules/model_evaluation.html#classification-metrics (fetched 2026-05-21)
- pandas 2.3.0 release notes: https://pandas.pydata.org/docs/whatsnew/v2.3.0.html (fetched 2026-05-21)
- Existing CSV schema: `/home/theo/AI/Projeto_2_AI/dados/archive/campeonato-brasileiro-full.csv` (read 2026-05-21)
- Existing pyproject.toml: `/home/theo/AI/Projeto_2_AI/pyproject.toml` (read 2026-05-21)
