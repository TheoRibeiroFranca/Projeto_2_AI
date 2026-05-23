---
phase: 02-feature-engineering
reviewed: 2026-05-23T12:36:14Z
depth: standard
files_reviewed: 1
files_reviewed_list:
  - notebook_data.ipynb
findings:
  critical: 2
  warning: 4
  info: 2
  total: 8
status: issues_found
---

# Phase 02: Feature Engineering — Code Review Report

**Reviewed:** 2026-05-23T12:36:14Z
**Depth:** standard
**Files Reviewed:** 1 (`notebook_data.ipynb`)
**Status:** issues_found

## Summary

`notebook_data.ipynb` implements data loading, cleaning, train/test splitting, and a feature
engineering pipeline (`build_features`) that produces rolling-window and season-cumulative
features for Brazilian league match prediction.

**The primary temporal-leakage guard (`shift(1).rolling(5, min_periods=5)`) is correctly
applied throughout.** An independent leakage verification cell (Section 11) confirms that
all first-match-per-team-season rows are NaN pre-imputation, as expected. The train/test
split is date-based (year <= 2022 / year >= 2023) with an assertion confirming zero
temporal overlap. No cross-season leakage was found in the rolling path.

Two critical issues were identified: (1) a silent correctness bug in `derive_result` that
can silently mislabel any match with a novel `vencedor` value as `AwayWin`, and (2) a same-day
double-header condition in the source CSV (31 occurrences) where the second match picks up
the first match's result inside the rolling window before it is shifted away — creating a
small but real leakage path. Four warnings were found covering code ordering risk, dead code,
a misleading assertion, and a missing identity assertion.

---

## Critical Issues

### CR-01: `derive_result` else-clause silently misclassifies unexpected `vencedor` values as AwayWin

**File:** `notebook_data.ipynb` — cell `3391c415`

**Issue:** The fallthrough `else` branch of `derive_result` returns `'AwayWin'` for any
`vencedor` value that is neither `'-'` nor exactly equal to `mandante`. This means any data
corruption, encoding inconsistency, or future name variant in the `vencedor` column that
does not exactly match the home team string will be silently labelled as an away win with
no error raised. The assertion immediately following only checks for nulls and the expected
three values — it would catch a fourth value but would NOT catch the case where a home-win
match was miscoded as away-win because `vencedor` had a slight name variant.

Current data has 0 unmatched rows, so this is latent. However, `canonical_names` is
intentionally designed to be populated in the future (cell `e2420af0`). If it is ever
populated, `derive_result` will still compare the **original** (pre-normalization) names,
while `vencedor` in the CSV may use a variant form that matches neither raw `mandante` nor
raw `visitante` — silently producing wrong `AwayWin` labels at scale.

**Fix:**
```python
def derive_result(row):
    if row['vencedor'] == '-':
        return 'Draw'
    elif row['vencedor'] == row['mandante']:
        return 'HomeWin'
    elif row['vencedor'] == row['visitante']:
        return 'AwayWin'
    else:
        raise ValueError(
            f"Unexpected vencedor '{row['vencedor']}' for match ID {row['ID']} "
            f"(mandante='{row['mandante']}', visitante='{row['visitante']}')"
        )
```

---

### CR-02: Same-day double-header in long-format table creates rolling-window leakage for 31 team-match pairs

**File:** `notebook_data.ipynb` — cell `fe10abc2`, `build_features`, Step 2

**Issue:** The source CSV contains at least 31 `(team, date)` pairs where a single team
appears twice on the same calendar date (once as home team, once as away team in a different
match). This is a data artifact from rescheduled fixtures. In the long-format table, these
rows are sorted by `['team', 'season', 'date', 'id']`. Because the two matches share the
same date, sort order is determined by `id`. When `shift(1).rolling(5)` is applied, the
second match (higher id) includes the shifted result of the first match (lower id) in its
window — before it has been shifted out. This means the rolling feature for the second
match reflects the outcome of a same-day match that was ongoing or just finished, which is
not strictly pre-match information.

Impact: 31 affected team-match pairs out of ~18,330 long-table rows (0.17%). The effect on
model accuracy is negligible, but the leakage verification in Section 11 does NOT detect
this case (it only checks for NaN in the first row per team-season, not for same-day
ordering issues).

**Fix:** Sort the long table with a tie-break on match `round` rather than `id`, or filter
out same-day duplicates with a diagnostic assertion:

```python
# Add assertion to detect and warn about same-day double-headers
team_date_counts = long.groupby(['team', 'date']).size()
double_headers = team_date_counts[team_date_counts > 1]
if len(double_headers) > 0:
    print(f"WARNING: {len(double_headers)} (team, date) pairs appear twice in long format "
          f"(rescheduled fixtures). Rolling features for these 2nd matches include "
          f"same-day match results. Affected pairs:\n{double_headers}")
```

For a complete fix, deduplicate or ensure consistent ordering by adding `round` to the sort
key in Step 2:
```python
long = pd.concat([...]).sort_values(['team', 'season', 'round', 'date', 'id']).reset_index(drop=True)
```

---

## Warnings

### WR-01: `derive_result` executes before team name normalization — design ordering risk

**File:** `notebook_data.ipynb` — cells `3391c415` (derive_result) then `e2420af0` (normalize)

**Issue:** `derive_result` compares `row['vencedor']` against `row['mandante']` using the
raw, un-normalized names from the CSV. Team name normalization via `canonical_names` runs in
the next cell (`e2420af0`), which maps `home_team`/`away_team` columns. The `vencedor`
column is never normalized and is dropped after `derive_result` runs. This ordering is
currently safe because `canonical_names = {}`. However, if `canonical_names` is ever
populated to normalize a team name that also appears in the `vencedor` column (e.g.,
`'Atletico MG'` → `'Atletico-MG'`), `derive_result` will compare the raw variant against
the raw home/away name — which still match — so it would still be correct for home/away
side matching. The real risk is the reverse: if `vencedor` in the CSV already uses the
canonical form but `mandante` uses the variant, `derive_result` would silently mislabel.

**Fix:** Move the normalization step before `derive_result`, or apply the same mapping
to `vencedor` before the result derivation:

```python
# Normalize vencedor with the same dict before derive_result
for col in ['mandante', 'visitante', 'vencedor']:
    df[col] = df[col].replace(canonical_names)

df['result'] = df.apply(derive_result, axis=1)
```

---

### WR-02: `points_last5` and `goal_diff_last5` computed before imputation loop is dead code

**File:** `notebook_data.ipynb` — cell `fe10abc2`, `build_features`, between Step 2 and Step 3

**Issue:** The two derived columns are computed immediately after the rolling step:

```python
# FEAT-04: derived columns computed before imputation
long['points_last5']    = long['win_last5'] * 3 + long['drw_last5']
long['goal_diff_last5'] = long['goals_scored_last5'] - long['goals_conceded_last5']
```

At this point `win_last5` and `drw_last5` contain NaN for early-season rows. These NaN
values propagate into `points_last5` and `goal_diff_last5`. Both columns are then
re-derived after the imputation loop (correctly). The first computation is dead: its
results are immediately overwritten. There is no code path that reads these intermediate
NaN-containing columns before the re-derivation.

**Fix:** Remove the first computation block entirely and keep only the post-imputation
derivation:

```python
# (Remove the pre-imputation block)
# After Step 3 imputation:
long['points_last5']    = long['win_last5'] * 3 + long['drw_last5']
long['goal_diff_last5'] = long['goals_scored_last5'] - long['goals_conceded_last5']
```

---

### WR-03: `is_monotonic_increasing` assertion is imprecise for the leakage it is intended to guard

**File:** `notebook_data.ipynb` — cell `c74dc3d5`

**Issue:** The assertion `assert df['date'].is_monotonic_increasing` uses pandas'
`is_monotonic_increasing`, which means **non-decreasing** (allows equal adjacent values).
The CSV has up to 15 matches on a single day (confirmed: 12 days with >10 matches), so this
assertion passes trivially. It provides no guarantee about the ordering of same-day matches —
only that dates do not go backwards. The comment implies this asserts that the data is
ordered for rolling computation, but same-day ordering is uncontrolled at this point.
`build_features` correctly handles this with `sort_values(['date', 'id'])`, making the
assertion a weak precondition check at best.

**Fix:** Rename the assertion comment to accurately describe what it checks, or add a
more informative diagnostic:

```python
# CSV dates are non-decreasing (equal dates allowed for same-day matches).
# build_features re-sorts by (date, id) to stabilize same-day order.
assert df['date'].is_monotonic_increasing, (
    "CSV is not in non-decreasing date order — check source file for corruption"
)
```

---

### WR-04: No assertion validates the `win + draw + loss` rolling sum identity

**File:** `notebook_data.ipynb` — cell `fe10abc3` and `fe10abc2`

**Issue:** After imputation and before saving, the integrity checks assert only that no NaN
values remain in the feature matrix. There is no assertion that
`home_wins_last5 + home_draws_last5 + home_losses_last5 == 5` for rows with a complete
rolling window (i.e., non-zero sum). This is a meaningful identity: for any team with
5+ prior season matches, the three outcome counts must sum to exactly 5. The imputation
for newly-promoted teams (no prior data) correctly produces all-zero rows (sum = 0), but
a bug in the imputation math (e.g., wrong scale factor) could produce inconsistent
non-5 sums that would pass the existing NaN assertion silently.

Manual verification in this review confirmed the identity holds (only 230 rows with sum=0,
all correctly from the 2003 inaugural season or newly-promoted teams), but no automated
guard exists.

**Fix:**
```python
# Add after the NaN assertion in the integrity check block:
wdl_sum = (fm['home_wins_last5'] + fm['home_draws_last5'] + fm['home_losses_last5'])
# Valid rows: sum is 5 (full window) or 0 (newly promoted / first season)
invalid = wdl_sum[(wdl_sum > 0.01) & (abs(wdl_sum - 5.0) > 0.01)]
assert len(invalid) == 0, (
    f"W+D+L rolling sum is not 0 or 5 in {len(invalid)} rows — imputation scale error"
)
```

---

## Info

### IN-01: `derive_result` uses row-wise `apply` — minor efficiency concern

**File:** `notebook_data.ipynb` — cell `3391c415`

**Issue:** `df.apply(derive_result, axis=1)` iterates row-by-row in Python. For 9,165 rows
this is fast enough (~100ms), but a vectorized approach using `np.select` would be idiomatic
pandas and significantly faster at scale.

**Fix:**
```python
import numpy as np
df['result'] = np.select(
    [
        df['vencedor'] == '-',
        df['vencedor'] == df['mandante'],
        df['vencedor'] == df['visitante'],
    ],
    ['Draw', 'HomeWin', 'AwayWin'],
    default='UNKNOWN'  # triggers assertion below
)
assert 'UNKNOWN' not in df['result'].values, "Unexpected vencedor value detected"
```

---

### IN-02: `h_n` denominator uses `hw` column as the count basis — implicit dependency

**File:** `notebook_data.ipynb` — cell `fe10abc2`, `build_features`, Step 4

**Issue:** The total number of prior home matches is computed as:

```python
home_m['h_n'] = (
    home_m.groupby(['home_team', 'season'])['hw']
    .transform(lambda x: x.shift(1).expanding().count())
)
```

`expanding().count()` counts non-NaN values. This works correctly because `hw` is always
`0.0` or `1.0` (never NaN), so count equals position-in-group minus one. However, using
a binary win-flag column as the vehicle for counting total matches is implicit and fragile:
if `hw` were ever NaN (e.g., due to upstream change in float encoding), the count would
undercount matches. A constant column would be more explicit.

**Fix:**
```python
home_m['_one'] = 1.0
home_m['h_n'] = (
    home_m.groupby(['home_team', 'season'])['_one']
    .transform(lambda x: x.shift(1).expanding().sum())
)
home_m.drop(columns=['_one'], inplace=True)
```

---

_Reviewed: 2026-05-23T12:36:14Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
