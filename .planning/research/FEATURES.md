# Feature Landscape

**Domain:** Brazilian football match outcome prediction (W/D/L classifier)
**Researched:** 2026-05-21
**Data source verified:** `dados/archive/` CSVs — 2003–2023, ~8,070 matches

---

## Data Availability Reality Check

Before categorizing features, what the CSVs actually provide:

| CSV | Contains | Usable Range |
|-----|----------|--------------|
| `campeonato-brasileiro-full.csv` | Match date, round, home/away team, score, winner, formation, coach, state | 2003–2023, fully populated |
| `campeonato-brasileiro-estatisticas-full.csv` | Shots, shots on target, possession %, passes, pass accuracy, fouls, cards, corners | **Mostly zero before ~2012–2013**; real data from ~round 1600+ (cross-check with IDs) |
| `campeonato-brasileiro-gols.csv` | Scorer, minute, goal type (penalty, own goal) per match | 2012 onwards (partial earlier) |
| `campeonato-brasileiro-cartoes.csv` | Card colour, player, position, minute per match | 2012 onwards (partial earlier) |

**Critical implication:** The scoring/results CSV is the only fully reliable source across all 20 years. Statistics (shots, possession) are zero-filled for a large portion of early years — using them as features will either require restricting training to ~2013+, or treating them as sparse/imputed. The ~65-70% accuracy target is achievable from results + goals alone (this is the published norm for European league studies too).

---

## Table Stakes

Features that every football prediction model must have. Missing any of these means the model cannot beat a trivial "always predict home win" baseline.

| Feature | Why Expected | Derivable From | Complexity | Notes |
|---------|--------------|----------------|------------|-------|
| **Home/away indicator** | Home advantage is the strongest single predictor in any league (~60% home wins baseline in Brazilian Série A) | `campeonato-brasileiro-full.csv` — `mandante` = home team | Low | Binary flag per team in row |
| **Goals scored, last N matches (home team)** | Most recent offensive form. Directly requested in PROJECT.md | Score columns + date sort + rolling window | Medium | N=5 is standard; requires chronological ordering per team |
| **Goals conceded, last N matches (home team)** | Most recent defensive form. Directly requested | Same | Medium | Must use opponent's score column, not own |
| **Goals scored, last N matches (away team)** | Symmetric | Same | Medium | |
| **Goals conceded, last N matches (away team)** | Symmetric | Same | Medium | |
| **Win/Draw/Loss streak, last 5 (home team)** | Momentum signal. Directly requested | Derived from `vencedor` column | Medium | Encode as numeric: W=1, D=0, L=-1 or one-hot per slot |
| **Win/Draw/Loss streak, last 5 (away team)** | Symmetric | Same | Medium | |
| **Home record (win%, draw%, loss%) — current season** | Team quality in home context | Filter by home role + season | Medium | Season must be inferred from date/round |
| **Away record (win%, draw%, loss%) — current season** | Team quality in away context | Filter by away role + season | Medium | |
| **Target: Home Win / Draw / Away Win** | Classification label | `vencedor` column, mapped to 0/1/2 | Low | Draw when `vencedor == "-"` |

**Minimum viable model uses only these features.** They are sufficient to approach 60-65% accuracy on held-out test, consistent with published sport prediction baselines. All are derivable from the primary CSV alone.

---

## Differentiators

Features that push accuracy from the 60-65% floor toward the 65-70% target or beyond. Not expected but high value.

| Feature | Value Proposition | Derivable From | Complexity | Notes |
|---------|-------------------|----------------|------------|-------|
| **Goal difference (last 5), not just raw goals** | Net goals (scored minus conceded) in rolling window captures quality of wins/losses, not just binary outcome | Score columns | Low incremental once goals computed | Add as a derived column after table stakes are built |
| **Points in last 5 matches** | Aggregates W/D/L into a single numeric (W=3, D=1, L=0) — more information-dense than separate streaks | `vencedor` | Low | Simple sum over last-5 results |
| **Head-to-head (H2H) record** | Fixture-specific history. Some rivalries (Fla-Flu, Atletico-Gremio) break expected form | Full CSV filtered by team pair | Medium | Rolling last 5 H2H; requires enough historical matches to avoid cold start (~2 seasons needed) |
| **Season position / points total at time of match** | Captures cumulative quality. A team on 45 pts plays differently than one on 15 pts | Cumulative from results within season year | High | Must reconstruct standing at each round — compute running standings table per season |
| **Days since last match** | Fatigue / rest effect. Brazilian schedule is extremely dense in Copa do Brasil periods | `data` column → date diff | Low | Simple date arithmetic |
| **Formation encoded (home/away)** | Tactical signal. 3-back systems vs 4-back correlates with defensive/offensive orientation | `formacao_mandante/visitante` — populated from 2014+ | Medium | Many nulls before 2014; treat as categorical, embed or one-hot top-N formations |
| **Coach stability** | Mid-season managerial changes correlate with short-term disruption | `tecnico_mandante/visitante` — populated later seasons | Medium | Binary: coach changed since last match? Requires lag |
| **Shots on target ratio (last 5)** | Better offensive quality signal than goals alone — smooths out lucky/unlucky results | `campeonato-brasileiro-estatisticas-full.csv` | Medium | Only reliable from ~2013+; do NOT backfill zeros — exclude or restrict to post-2013 subset |
| **Away goals per game — season** | Captures away team's specific road-scoring capability, separate from overall form | Score columns, filtered by away role | Low incremental | Already computed if table-stakes home/away records are built |
| **State (geographic region) — home team** | Altitude and climate effects exist in Brazilian football (Cuiaba, Belem heat; Belo Horizonte altitude). Captures home ground characteristics beyond team identity | `mandante_Estado` | Low | Encode as categorical; diminishing returns unless model is already strong |

**Recommended differentiator priority for reaching 65-70%:**
1. Goal difference + points-in-last-5 (trivially cheap, high signal)
2. H2H record (medium effort, known Série A effect)
3. Season standing at match time (high effort, meaningful signal for mid/late season)

---

## Anti-Features

Things to deliberately not build. Each has a clear reason.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **Player-level data (squad, injuries, suspensions)** | Not in the CSVs. External APIs (Football-Data.org, API-Football) charge for squad-level data; scrapers break constantly. Adds weeks of pipeline work for marginal gain on W/D/L (player effects are smoothed out by team averages over 5 matches) | Rely on team-level rolling form as a proxy |
| **xG (expected goals)** | Not in CSVs, requires Statsbomb/Opta license or Understat scraping. The statistics CSV has shots/shots-on-target which correlates with xG; use those as a cheaper proxy for later seasons | Use shots on target rolling average instead |
| **Market odds / betting lines** | Project.md explicitly scopes this out as "real-time betting integration." Also: odds are future-leaking if not handled carefully in training pipeline | Ignore |
| **Score prediction (exact scoreline or goals range)** | PROJECT.md scopes this out explicitly. Ordinal W/D/L is the target | Stay with 3-class classification |
| **Transfer market values (Transfermarkt)** | Requires external scraping of third-party site, updates seasonally, introduces significant data alignment complexity | Season position is a sufficient proxy for squad quality |
| **Weather data** | Brazilian football is predominantly played in covered/open tropical stadiums. Weather signal is weak for match outcomes — not worth the API + alignment cost | Home state encodes climate broadly |
| **Possession %, pass accuracy as primary features** | Available from 2021 dataset but zero-filled before 2013 — ~10 years of training data becomes unusable if these are required columns. High missingness | Use only if restricting to post-2013 and only as secondary differentiators |
| **Per-match card counts as direct features** | Cards are partly outcome-dependent (losing teams get more cards in stoppage time). Using card totals from past matches as predictors introduces mild label leakage semantics, and the signal is mostly captured by the W/D/L + goals features already | Skip unless adding disciplinary risk as a secondary signal |

---

## Feature Dependencies

```
mandante_Placar, visitante_Placar (score columns) — base of almost everything
  └── goals_scored_last5_home        (rolling, chronological sort per team)
  └── goals_conceded_last5_home      (rolling, opponent's score)
  └── goals_scored_last5_away
  └── goals_conceded_last5_away
  └── goal_diff_last5_home           (derived: scored - conceded)
  └── goal_diff_last5_away
  └── points_last5_home              (derived: W=3, D=1, L=0)
  └── points_last5_away

vencedor (winner column) — base of streak features
  └── wdl_streak_last5_home          (sequence of results, per team)
  └── wdl_streak_last5_away
  └── home_win_pct_season            (running per season, home role only)
  └── home_draw_pct_season
  └── home_loss_pct_season
  └── away_win_pct_season
  └── away_draw_pct_season
  └── away_loss_pct_season

vencedor × (mandante == visitante) — head-to-head
  └── h2h_record_last5               (requires team pair filter)

data (date column) — prerequisite for rolling windows
  └── chronological sort is MANDATORY before any rolling feature
  └── days_since_last_match (differentiator)

formacao_mandante / formacao_visitante
  └── formation_home_encoded         (nullable; 2014+ reliable)
  └── formation_away_encoded

mandante_Estado / visitante_Estado
  └── home_team_state                (static categorical)
```

**Hard dependency: chronological ordering per team.** All rolling features (last-5 goals, streak, home record) require matches to be sorted by date and then grouped by team. A mistake here (e.g., using `rodata` round number as a proxy for time without date) causes silent feature contamination with future data.

---

## MVP Recommendation

For the stated goal — ~65-70% accuracy, clean notebook, academic project — build exactly this and nothing more:

**Priority 1 — Table stakes (required for any meaningful model):**
1. `is_home_team` flag (each row has home-team perspective)
2. Goals scored/conceded last 5 (rolling, both teams)
3. W/D/L result in each of last 5 matches (one-hot 5×3 or numeric sequence)
4. Home win/draw/loss % current season (home team)
5. Away win/draw/loss % current season (away team)

**Priority 2 — Cheap differentiators (add before first training run):**
6. Goal difference last 5 (derived from #2, zero effort)
7. Points last 5 (derived from #3, zero effort)

**Priority 3 — If accuracy falls below 63% after Priority 1-2:**
8. Head-to-head last 5 meetings

**Defer indefinitely:**
- Season standings table (high effort, unclear ROI for academic submission)
- Shot/possession features (data coverage gap makes them risky)
- Formation, coach change (sparse data, marginal signal)

**Target training data window:** Use all seasons (2003–2023). Drop only the first 5 matches of each team's season (insufficient history for rolling window). This yields ~7,500+ usable match rows — sufficient for random forest, gradient boosting, or a small MLP.

---

## Accuracy Expectation Calibration

- **Naive baseline** (always predict home win): ~45-50% in Brazilian Série A (home advantage is real but weaker than England/Spain)
- **Historical home/away win% only**: ~53-57%
- **Full table-stakes feature set + any ML model**: ~60-65%
- **Table stakes + priority differentiators**: ~65-70%
- **65-70% is the published ceiling for open-data football prediction** — this is not a model quality failure, it is intrinsic sport unpredictability. Draws are particularly hard to predict (random forest commonly misclassifies ~40-50% of draws).

The 65-70% target in PROJECT.md is realistic and achievable with the feature set above. Claiming higher without proprietary data (Opta, Statsbomb, betting odds) would require luck or overfitting.

---

## Sources

- Data structure verified directly from `dados/archive/*.csv` and `Legenda.txt` (HIGH confidence)
- Feature prioritization based on football prediction ML literature patterns — Poisson regression, Dixon-Coles, ELO-based models, random forest approaches on structured match data (MEDIUM confidence — cannot verify directly due to tool restrictions)
- 65-70% accuracy ceiling: widely cited in sport prediction literature for open-data W/D/L classifiers (MEDIUM confidence)
- Brazilian home advantage estimate (~45-50% home wins vs European ~55-60%): based on domain knowledge of Série A volatility (LOW confidence — verify against actual CSV win distribution before designing class-weight strategy)
