---
phase: 01-data-ingestion-cleaning
reviewed: 2026-05-22T00:00:00Z
depth: standard
files_reviewed: 3
files_reviewed_list:
  - notebook_data.ipynb
  - pyproject.toml
  - pytest.ini
findings:
  critical: 2
  warning: 3
  info: 2
  total: 7
status: issues_found
---

# Phase 01: Code Review Report

**Reviewed:** 2026-05-22
**Depth:** standard
**Files Reviewed:** 3
**Status:** issues_found

## Summary

Three files were reviewed: the Phase 1 data pipeline notebook (`notebook_data.ipynb`), the project dependency manifest (`pyproject.toml`), and the pytest configuration (`pytest.ini`). The notebook correctly loads, cleans, and splits the Brazilian Série A dataset; all core assertions pass against the actual data. However, two blockers were found: the `season` column silently mislabels 185 matches from the 2020 competition as `season=2021` (the COVID-delayed 2020 Brasileirão ran into January–February 2021), and the `pytest.ini` configuration runs the legacy MNIST notebook which has a known undefined-variable crash (`x_sint`), making the test suite non-functional. Three additional warnings cover missing round-trip validation for the test parquet, `scikit-learn` absent from `pyproject.toml`, and the `ID` column casing mismatch against the documented schema contract.

---

## Critical Issues

### CR-01: `season` column silently mislabels 2020 championship matches as season 2021

**File:** `notebook_data.ipynb` — cell `fa8b0abb` (season derivation via `df['date'].dt.year`)

**Issue:** The 2020 Brasileirão was suspended by COVID and ran from August 2020 through February 2021. Rounds 28–38 (185 matches) were played in January–June 2021. Because `season = date.dt.year` assigns the calendar year, those 185 matches receive `season=2021` instead of `season=2020`. The impact is concrete and measurable:

- `season=2020` in the parquet contains only 268 rows (rounds 1–27) — an incomplete, truncated championship.
- `season=2021` contains 492 rows — two different championships interleaved, with rounds 1–38 duplicated.

Any downstream model notebook that groups by season, filters by season, or uses season as a stratification key will silently operate on malformed per-season groups. Rolling-window form features spanning the 2020/2021 boundary will not benefit from season labels that correctly identify competition membership. No assertion in the notebook catches this condition.

**Fix:** Assign the championship year based on the competition, not the calendar year. Since the Brasileirão always starts mid-year, a robust heuristic is: if `month < 5`, assign the previous year as the season. Alternatively, accept the calendar-year proxy but add an explicit assertion that no season group exceeds 400 rows:

```python
# Option A: month-based heuristic
df['season'] = df['date'].apply(
    lambda d: d.year - 1 if d.month < 5 else d.year
)

# Option B: keep calendar year but add guard assertion
for yr, grp in df.groupby('season'):
    assert len(grp) <= 420, (
        f"season={yr} has {len(grp)} rows — likely mixing two championships "
        f"(expected max ~400 for a 20-team season)"
    )
```

Option A is preferred because it produces semantically correct season labels. Verify the resulting `season=2020` group has 453 rows (rounds 1–38) and `season=2021` has 380 rows.

---

### CR-02: `pytest.ini` causes test suite to always fail by executing the broken legacy notebook

**File:** `pytest.ini` — lines 1–4

**Issue:** `testpaths = .` combined with `addopts = --nbmake -x` makes pytest discover and execute every `.ipynb` file under the project root that is not in `norecursedirs`. Both `notebook_data.ipynb` and `notebook.ipynb` are collected (confirmed with `--collect-only`). The legacy MNIST notebook (`notebook.ipynb`) has a known undefined variable `x_sint` / `y_sint` documented in `CLAUDE.md` under "Anti-Patterns — Undefined Variable Reference". Execution will raise a `NameError` and, because `-x` is set (fail-fast), pytest will stop without running `notebook_data.ipynb`. The CI test command documented in `CLAUDE.md` (`pytest --nbmake notebook_data.ipynb -x`) bypasses this by passing an explicit path, but the default `pytest` invocation is broken.

**Fix:** Restrict the default test target to `notebook_data.ipynb` only:

```ini
[pytest]
addopts = --nbmake -x
testpaths = notebook_data.ipynb
norecursedirs = .venv .git __pycache__ .planning
```

Or add `notebook.ipynb` to `norecursedirs` equivalent via `collect_ignore` in a `conftest.py`:

```python
# conftest.py
collect_ignore = ["notebook.ipynb"]
```

---

## Warnings

### WR-01: No round-trip validation for `matches_test.parquet`

**File:** `notebook_data.ipynb` — cell `cc8ce8c6`

**Issue:** The save cell reads back `matches_train.parquet` and asserts row count and column names. It does not perform an equivalent check for `matches_test.parquet`. The only validation for the test file is `os.path.exists(...)`. A silent write failure (wrong schema, truncated rows, wrong dtypes) on the test file would pass all assertions. All three model notebooks (Phases 2–4) load from both parquet files; a corrupt test set would silently produce misleading evaluation results.

**Fix:** Add a symmetric round-trip check for the test file after the existing train check:

```python
reload_test = pd.read_parquet('dados/matches_test.parquet')
assert len(reload_test) == 1140, f"Round-trip test: esperado 1140, obtido {len(reload_test)}"
assert list(reload_test.columns) == expected_cols, (
    f"Colunas inesperadas no test parquet: {list(reload_test.columns)}"
)
```

---

### WR-02: `scikit-learn` absent from `pyproject.toml` dependencies

**File:** `pyproject.toml` — lines 6–19

**Issue:** `scikit-learn` is the core dependency for all three model notebooks (Phases 2–4). It is not declared in `pyproject.toml`. `CLAUDE.md` notes it must be added manually via `uv add scikit-learn`. A fresh environment created from `pyproject.toml` alone (e.g., CI, a new contributor) will silently lack `scikit-learn`; the model notebooks will fail at the first `from sklearn` import with no indication that the fix is a dependency declaration rather than a code error. This is a reproducibility gap.

**Fix:** Add `scikit-learn` to `pyproject.toml` dependencies:

```toml
dependencies = [
    ...
    "scikit-learn>=1.3",
    ...
]
```

---

### WR-03: Output schema contract violates `CLAUDE.md`: `ID` column is uppercase

**File:** `notebook_data.ipynb` — cell `fa8b0abb` (`keep_cols` definition)

**Issue:** `CLAUDE.md` documents the parquet output schema as 11 columns with lowercase names: `id, date, season, round, ...`. The notebook's `keep_cols` list and the round-trip assertion both use `'ID'` (uppercase). The parquet files on disk have `'ID'`. Any model notebook that references `df['id']` per the documented schema will raise a `KeyError`. The contract inconsistency will surface when new notebooks are written following the documentation.

**Fix:** Either rename the column to lowercase in the selection step:

```python
df = df.rename(columns={'ID': 'id', ...})
keep_cols = ['id', 'date', 'season', 'round', ...]
```

Or update `CLAUDE.md` to reflect `ID` (uppercase) as the canonical name. Renaming to lowercase is preferred for consistency with the rest of the schema.

---

## Info

### IN-01: `FigureCanvasAgg` return value unused — pattern is correct but non-idiomatic

**File:** `notebook_data.ipynb` — cell `57d532db`

**Issue:** `FigureCanvasAgg(fig)` is called but its return value is not assigned. This works because `FigureCanvasAgg.__init__` attaches itself to `fig.canvas` as a side effect. The pattern is correct for headless rendering but is unconventional and may confuse readers who expect a canvas variable to be used later.

**Fix:** Assign to a variable to make the intent explicit, or use `matplotlib.figure.Figure` with the `canvas_class` parameter:

```python
canvas = FigureCanvasAgg(fig)
# ... render ...
canvas.print_figure(buffer, format='png', bbox_inches='tight', dpi=120)
```

---

### IN-02: `canonical_names = {}` provides no normalization — team name variants will silently pass

**File:** `notebook_data.ipynb` — cell `e2420af0`

**Issue:** The normalization dict is empty with a comment stating "CSV names are already consistent." This is verified as true for the current dataset. However, the guard assertion (`team_counts.get(club, 0) >= 380`) would catch any future CSV update that introduces new name variants for top clubs below the 380-match threshold — but only for the 10 listed clubs. New seasons added to the CSV with variant spellings for other clubs (e.g., a promoted team with a historical name change) would pass all assertions silently. The empty dict with no-op replace is dead code as shipped.

**Fix:** If no normalization is currently needed, remove the replace call and add a comment documenting the decision and how to populate the dict when variants appear. The current form creates a false impression that normalization is active:

```python
# Team names in this dataset are consistent across all 23 seasons (verified 2026-05).
# If future CSV updates introduce variants, add mappings here:
# canonical_names = {'Atletico MG': 'Atletico-MG'}
# for col in ['home_team', 'away_team']:
#     df[col] = df[col].replace(canonical_names)
```

---

_Reviewed: 2026-05-22_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
