---
title: "Backed-Depth Divergence Candidate Slate 2026-07-06 v2 (FROZEN)"
type: "report"
date: "2026-07-06"
status: "FROZEN -- slate frozen before any paid generation"
project: "DAI"
slice: "Backed-Depth Divergence Capture (PAID) v2"
tags:
  - calibration
  - cohort
  - backed-depth
  - divergence
  - capture
  - slate
related:
  - "06 Execution/reports/backed-depth-divergence-capture-readiness-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
---

# Backed-Depth Divergence Candidate Slate 2026-07-06 v2 (FROZEN)

## freeze statement

**Frozen at 2026-07-06 08:36:51 EDT (12:36:51Z).** No paid model call was made, and
`agent-service` was not started for generation, before this freeze. Screening used only
read-only StatsAPI, one the-odds-api h2h slate read, and the read-only
`GET /api/agent-runs/source-readiness` app path (no model, no db write). AgentRuns count
at freeze = **273** (unchanged from the 2026-07-05 v1 baseline). Outcomes=112,
Evaluations=112.

After this freeze: no game is added because early outputs agree with market, no generated
game is removed because it is inconvenient, and all failures are recorded explicitly.

## caps (binding)

- MAX_PAID_RUNS = 12
- TOTAL_COST_CAP_USD = 0.05
- MAX_MODEL_CALLS_PER_RUN = 1
- MODEL_EXPECTED = gpt-4o-mini
- SETTLEMENT_IN_THIS_SLICE = false

## timing rationale (why generate before the 10:00 ET target)

Freeze/screen ran at ~08:37 ET, ~1h20m before the 10:00-13:00 ET target window. The
window exists to guarantee (a) pre-game status and (b) a posted, book-backed market. Both
are satisfied now with large margin: all 8 games are `Scheduled` (pre-game), earliest
first pitch 14:10 ET (~5.5h out), and every game returns `market.level=backed_depth` with
9 books. Phase 1 of the slice explicitly permits generation outside the window when >= 4
suitable pre-game candidates remain -- 6 remain. Running early strictly avoids the v1
failure mode (too late; slate exhausted). There is no market-maturity or pre-game downside
to generating now versus at 10:00.

## slate screening method

1. StatsAPI schedule (`sportId=1&date=2026-07-06`) -> 8 games, all pre-game.
2. One the-odds-api `baseball_mlb h2h` slate read (us region, american odds; 1 quota unit,
   402 remaining after). Per game: de-vigged implied prob per book, averaged across books;
   favorite = higher mean prob; implied-probability gap = |P(home) - P(away)|.
3. `GET /api/agent-runs/source-readiness` per game (app path; no model, no db write; each
   triggers one the-odds-api unit) -> identity, starter level, market level/books,
   predicted observed data regime, eligibility for `starter_enriched_market_backed_depth`.

## divergence prefilter

- Primary: pre-game, identity-safe, backed_depth source-ready, market baseline available,
  sufficient books, implied-probability gap `<= ~0.10` (close favorite / mixed signal).
- Secondary: gap `> 0.10` and `<= 0.15` (none on this slate).
- Exclude: gap `> 0.15` (overwhelming favorite), or not pre-game / source-thin / no
  identity / no market baseline.

## all 8 games considered

| gamePk | matchup (away @ home) | start (ET) | status | pre-game | identity | starter | market | books | consensus | mkt favorite | favP (de-vig) | gap | tier | decision |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 823205 | Toronto Blue Jays @ San Francisco Giants | 21:45 | Scheduled | yes | matched | enriched | backed_depth | 9 | away | Toronto Blue Jays | 0.508 | 0.017 | primary | **SELECTED** |
| 823282 | Arizona Diamondbacks @ San Diego Padres | 21:40 | Scheduled | yes | matched | enriched | backed_depth | 9 | home | San Diego Padres | 0.509 | 0.019 | primary | **SELECTED** |
| 822958 | New York Yankees @ Tampa Bay Rays | 18:40 | Scheduled | yes | matched | enriched | backed_depth | 9 | home | Tampa Bay Rays | 0.514 | 0.027 | primary | **SELECTED** |
| 823036 | Milwaukee Brewers @ St. Louis Cardinals | 19:45 | Scheduled | yes | matched | enriched | backed_depth | 9 | away | Milwaukee Brewers | 0.521 | 0.042 | primary | **SELECTED** |
| 822712 | Houston Astros @ Washington Nationals | 18:45 | Scheduled | yes | matched | enriched | backed_depth | 9 | home | Washington Nationals | 0.522 | 0.043 | primary | **SELECTED** |
| 824900 | New York Mets @ Atlanta Braves | 19:15 | Scheduled | yes | matched | enriched | backed_depth | 9 | home | Atlanta Braves | 0.545 | 0.090 | primary | **SELECTED** |
| 824089 | Philadelphia Phillies @ Kansas City Royals | 14:10 | Scheduled | yes | (not screened) | - | - | 9 | - | Philadelphia Phillies | 0.660 | 0.319 | exclude | EXCLUDED |
| 823930 | Colorado Rockies @ Los Angeles Dodgers | 22:10 | Scheduled | yes | (not screened) | - | - | 9 | - | Los Angeles Dodgers | 0.662 | 0.324 | exclude | EXCLUDED |

Start times are converted from StatsAPI UTC to ET; the app reports all as game date
2026-07-06 (local).

## selected cohort (6 close-favorite backed_depth games)

Each is identity-safe (StatsAPI gamePk matched), starter-enriched, market `backed_depth`
with 9 books, and `generationEligibleForTargetRegime=true` for
`starter_enriched_market_backed_depth`. The source-readiness consensus side matches the
the-odds-api favorite on all six, so the market baseline is internally consistent.

Why each is useful for divergence measurement: all six are close favorites (gap 0.017 to
0.090), where the market itself is near-indifferent. If DAI's lean lands on the underdog --
or on the favorite with materially different confidence -- that is readable DAI-vs-market
divergence, which is exactly the evidence Gate 4 needs (it currently fails on
`insufficient_market_disagreement` and `insufficient_market_coverage`). Games this close
are the highest-information places to look for an independent signal.

## exclusions

- 824089 PHI @ KC -- EXCLUDED: overwhelming favorite (PHI implied 0.660, gap 0.319 > 0.15).
- 823930 COL @ LAD -- EXCLUDED: overwhelming favorite (LAD implied 0.662, gap 0.324 > 0.15).

Both were left un-screened for source-readiness on purpose (excluded on market gap before
spending source-readiness quota); both are pre-game with 9 books but fail the divergence
prefilter as near-certain markets where DAI-market agreement carries little signal.

## pre-generation baselines (for post-slice validation)

- AgentRuns = 273
- AgentRunOutcomes = 112
- AgentRunEvaluations = 112
- Existing active runs for the 6 target gamePks = 0 (no duplicate risk)
- the-odds-api quota remaining after screening = 402 (used: 1 slate read + 6 source-readiness = 7 units)
