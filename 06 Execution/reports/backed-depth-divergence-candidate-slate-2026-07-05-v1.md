---
title: "Backed-Depth Divergence Candidate Slate (frozen) 2026-07-05 v1"
type: "report"
date: "2026-07-05"
status: "frozen -- slate exhausted; 0 close-favorite divergence candidates; paid generation BLOCKED pre-spend"
project: "DAI"
slice: "Backed-Depth Divergence Capture (PAID) v1 -- slate freeze"
repos:
  dai: "unchanged (c6d4f43)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - backed-depth
  - divergence
  - capture
related:
  - "06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-05-v1.md"
---

# Backed-Depth Divergence Candidate Slate (frozen) 2026-07-05 v1

**freeze timestamp:** 2026-07-05T21:44Z (17:44 ET). **caps:** MAX_PAID_RUNS=12, TOTAL_COST_CAP_USD=0.05,
gpt-4o-mini, 1 model call/run. **candidate mode:** NAMED_GAMEPKS=auto (game-day slate). **outcome:** slate frozen;
**no game passes the divergence prefilter**; paid generation blocked before any model call.

## slate discovery (StatsAPI schedule, free)

Official MLB slate for 2026-07-05 = 15 games. Status at freeze time (21:44Z / 17:44 ET):

- **Final (7):** 824902 NYM@ATL, 822715 PIT@WSH, 824497 BAL@CIN, 823525 MIN@NYY, 824656 STL@CHC, 824091 PHI@KC, (and 824413/others below moved to in-progress).
- **In Progress (6):** 824413 CWS@CLE, 824172 TB@HOU, 822879 DET@TEX, 824333 SF@COL, 825061 MIL@AZ, 824980 MIA@ATH, 823117 TOR@SEA.
- **Pre-Game (2):** 823931 SD@LAD (23:20Z), 824010 BOS@LAA (2026-07-06T01:30Z).

Only the 2 Pre-Game games are candidates for pre-game backed_depth generation; the 13 Final/In-Progress games are ineligible (generation must be pre-first-pitch to capture a pre-game market + non-contaminated read).

## screening of the 2 pre-game candidates

Screened read-only via the app path `GET /api/agent-runs/source-readiness` (mirrors generation retrieval; no model call, no write) plus one the-odds-api h2h read for implied probabilities.

| gamePk | matchup | identity | starter | market | books | favorite | home_impl | away_impl | gap | eligible(backed_depth) |
|---|---|---|---|---|---|---|---|---|---|---|
| 823931 | SD @ LAD | matched | enriched (Sears/Sheehan) | backed_depth | 9 | home (LAD) | 0.690 | 0.346 | **0.344** | yes |
| 824010 | BOS @ LAA | matched | enriched (Suarez/Johnson) | backed_depth | 9 | away (BOS) | 0.426 | 0.613 | **0.188** | yes |

Both are identity-safe, enriched-starter, 9-book backed_depth, and generation-eligible for
`starter_enriched_market_backed_depth`. So the *regime* is available -- the block is purely the divergence
prefilter.

## divergence prefilter applied (honestly)

Prefilter target (from the plan): prefer close favorites, implied-probability gap `<= ~0.10`; **avoid overwhelming
favorites**; seek games where DAI is not mechanically pulled to the market favorite.

- **823931 SD@LAD -- EXCLUDED.** LAD is an overwhelming favorite (home implied 0.690, gap 0.344). The plan
  explicitly says avoid overwhelming favorites. Not a close-favorite divergence candidate.
- **824010 BOS@LAA -- EXCLUDED (marginal).** BOS favorite at 0.613, gap 0.188 -- a moderate, not close, favorite;
  ~2x the `<=0.10` target. A single marginal game is not a divergence cohort and does not justify a paid slice.

**Selected for generation: 0 games.** No surviving candidate meets the close-favorite divergence criterion.

## decision: BLOCK paid generation (PARTIAL)

The 2026-07-05 slate is exhausted down to 2 pre-game games, and neither is a close-favorite divergence candidate.
Generating them would only add likely market-agreement rows (the exact limitation the prior 7-game cohort already
has) -- it would not advance the divergence-measurement premise this slice exists to serve. Per the plan
("the prefilter is allowed to seek measurement value, not a desired result") and the operator prompt (which lists
"PARTIAL: capture blocked before paid generation" as a valid outcome and says to avoid overwhelming favorites), the
correct action is to **stop before spending**.

No paid model call was made. agent-service was never started. Registry routing stayed default-off (canary env never
set). AgentRuns count unchanged at 273.

## why the slate was thin (root cause + fix)

The capture ran at 17:44 ET, by which point 13 of 15 games were already Final/In-Progress. Pre-game backed_depth
generation needs games a few hours before first pitch with starters confirmed and odds posted. **Fix: run the
capture mid-to-late morning ET (e.g. 10:00-13:00 ET)**, when the full evening slate is still pre-game and markets
are posted -- that window offers ~10-15 pre-game candidates to apply the divergence prefilter against.

## next slice

Reschedule the paid divergence capture to a game day, executed 10:00-13:00 ET, screening the full pre-game slate
via `/source-readiness` + a single the-odds-api slate read, selecting close favorites (gap `<= ~0.10`) first. Keep
all other guardrails identical.
