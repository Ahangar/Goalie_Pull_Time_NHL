# NHL Goalie Pull Analysis — Conversation Summary

## Overview
This conversation covered a review of Faraz Ahangar's Medium article and codebase analyzing goalie pull timing in the NHL, with a discussion of turning the work into a scientific paper.

---

## 1. Article Review
**Source:** [Goalies Are Being Pulled Earlier in NHL; Does It Pay off?](https://medium.com/@Faraz_EA/goalies-are-being-pulled-earlier-in-nhl-does-it-pay-off-465472e6c321)

### Key Findings
- Average goalie pull time grew from ~1.5 minutes (2011–12) to ~2.5 minutes (2021–22)
- Most pulls are unsuccessful — the chance of conceding is 2.6x higher than scoring
- Success rate modestly improved from 0.11 to 0.15 over the decade
- Optimal pull time estimated at **4.5 minutes** before the end of the game
- A notable methodological contribution: explicit handling of **delayed penalties**, which cause spurious early goalie pull detections

---

## 2. Path to Publication

### Existing Literature
| Paper | Key Finding |
|---|---|
| Beaudoin & Swartz (SFU) | Pull 5–8 min remaining depending on puck location; incorporates penalties |
| Brown & Asness (2018, NYU/AQR) | Optimal pull at 6:10 remaining — ~3x earlier than convention |
| Alex Galea (Medium) | Optimal pull at ~3:00 remaining; noted delayed penalty issue |
| Meghan Hall | 18.1% success rate in 2020–21 for teams down one goal |

### Unique Contributions of This Work
- Decade-long longitudinal trend (2011–2022)
- Explicit delayed penalty handling
- Combining trend analysis with optimal pull time in one study

### Recommended Journals
1. **Journal of Quantitative Analysis in Sports (JQAS)** — best fit; ASA official journal, covers within-game strategy, open access from 2026 with no author fees
2. **Statistical Analysis and Data Mining (ASA Data Science Journal)** — runs sports analytics special issues
3. **Journal of Sports Analytics** — applied focus, receptive to hockey work
4. **MIT Sloan Sports Analytics Conference** — best venue for visibility

### What's Needed for Publication
- Formal model (logistic regression or survival analysis)
- Confidence intervals on success rate estimates
- Score differential breakdown (down 1 vs. down 2)
- Literature review section
- Leverage GitHub repo for reproducibility

---

## 3. Code Review (4 Notebooks)

### Step 0 — HTML Download
- **Good:** `random_wait()` using Beta distribution avoids rate-limiting
- **Issue:** Typo `url_tempalte` (missing 'a')
- **Issue:** Loop only covers one season; needs clean multi-season loop for paper
- **Suggestion:** Skip already-downloaded files to avoid re-scraping

### Step 1 — HTML Parsing to DataFrame
- **Good:** Away/home line separation using `is_away` flag is clean
- **Issue:** Time parsing (`x.split(':')[1][2:]`) is brittle across seasons — needs try/except
- **Issue:** Silently falls back to `['', '']` on team name extraction failure — should log instead

### Step 2 — Goalie Pull Detection
- **Good:** Delayed penalty filter is the key methodological contribution
- **Issue:** `~` should be used instead of `-` for boolean NOT in pandas
- **Issue:** `get_goalie_pulls()` was called twice per game — doubles runtime
- **Issue:** Only the first pull per game is captured — multiple pulls not handled

### Step 3 — Analysis
- **Issue:** `pd.cut(..., 18)` bin boundaries depend on data range — should be explicit
- **Issue:** `order=2` polynomial fit not justified — compare degrees and use AIC/BIC
- **Issue:** Optimal time read visually from plot — should be computed numerically
- **Missing:** No confidence intervals on binned success rates
- **Missing:** No score differential breakdown

---

## 4. Step 2 Fixes Applied

### Fix 1: Eliminate Double Call to `get_goalie_pulls()`
**Before:**
```python
if get_goalie_pulls(game_data) is not None:
    game_pull_time, game_success = get_goalie_pulls(game_data)
```
**After:**
```python
result = get_goalie_pulls(game_data)
if result is not None:
    game_pull_time, game_success = result
```
**Impact:** Cuts runtime from ~8 minutes to ~4 minutes.

---

### Fix 2: Multiple Goalie Pulls Per Game

Two options were discussed:

**Option A — Diagnostic only (recommended first step)**

Add a new cell after the main loop to measure how often multiple pulls occur:
```python
multiple_pulls = 0
total_pulls = 0

for file in os.listdir('./data/game/'):
    game_data = pd.read_csv('./data/game/' + file)
    game_data['total_sec'] = game_data['minute'].astype(int)*60 + game_data['second'].astype(int)
    game_data['n_row'] = game_data['n_row'].astype(int)
    game_data.loc[game_data['away_line'].isna(), 'away_line'] = 'G'
    game_data.loc[game_data['home_line'].isna(), 'home_line'] = 'G'
    
    goalie_out = game_data.loc[
        (~game_data['away_line'].str.contains('G') | ~game_data['home_line'].str.contains('G')) &
        (game_data['total_sec'] < 360) & (game_data['period'] == 3)
    ]
    
    if len(goalie_out) > 1:
        gaps = goalie_out['n_row'].diff() > 5
        if gaps.any():
            multiple_pulls += 1
    
    if len(goalie_out) > 0:
        total_pulls += 1

print(f'Total games with a goalie pull: {total_pulls}')
print(f'Games with multiple pull events: {multiple_pulls}')
print(f'Percentage: {round(100 * multiple_pulls / total_pulls, 1)}%')
```

**Decision rule:**
- If < ~5% → acknowledge as a limitation in the paper
- If > ~5% → implement Option B (capture all pull events per game)

**Option B — Capture all pull events** (implement only if Option A shows high frequency)

Refactor `get_goalie_pulls()` to return a list of `[pull_time, success]` pairs, using `n_row.diff() > 5` to segment distinct pull events via a `pull_id` column.

---

### How the Diagnostic Works
| Code | Purpose |
|---|---|
| `total_sec < 360` | Limit to last 6 minutes of 3rd period |
| `~str.contains('G')` | Detect rows where a team has no goalie |
| `n_row.diff() > 5` | Gap in event numbers signals goalie returned to net |
| `gaps.any()` | At least one gap = multiple distinct pull events |

**Note:** The diagnostic does not apply the delayed penalty filter, so it may slightly overcount — acceptable for a quick diagnostic.
