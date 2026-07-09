---
title: "V2 Accelerated Capture -- Day 1 report (2026-07-09)"
type: "report"
date: "2026-07-09"
status: "COMPLETE -- 8/8 captured in window, 0 hard stops"
project: "DAI"
slice: "V2 Accelerated Capture Cadence v1 -- Day 1"
related:
  - "06 Execution/plans/v2-accelerated-capture-cadence-2026-07-09-v1.md"
  - "06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md"
  - "04 Products/sports-v1/prompting/prompt-market-context-hardening-v1.md"
---

# v2 accelerated capture -- day 1 (2026-07-09)

executed from the pushed runbook `06 Execution/plans/v2-accelerated-capture-cadence-2026-07-09-v1.md`
(per-day procedure, day 1 = no settlement pairing). window honored: pre-flight 07:34 ET, capture
10:12-10:21 ET (proven 10:00-13:00 ET window; session cron fired 10:12 ET as the manual-fallback
replacement for the dead 07-08 remote-session cron).

## operator directive applied (2026-07-09 morning, binding both days)

if fewer than 8 games pass eligibility, capture only the eligible set -- no backfilling volume with
weaker rows; the 8/day cap is a ceiling, not a target. (day 1: 10 eligible, so the cap bound; the two
dropped rows were the WIDEST-gap eligible games, dropped by rank, not by relaxation.)

## starting state

- dai: clean at `ce8f21f` (hardening commit = HEAD; only artifact = line-endings-only phantom
  `DevCore.Data.csproj`, empty diff). NOT touched today; nothing committed.
- dai-vault: runbook commit `a17b925` present; 2 pre-existing untracked files left alone.
- services cold at 07:34 ET: brought up docker desktop -> `devcore-sql` container -> DevCore.Api :5007
  (Development). agent-service kept DOWN until capture.
- duplicate-active sweep at pre-flight AND at window-open: **0** (247 active / 286 total).
- none of today's 13 gamePks had an existing active run.

## slate eligibility (13 games screened via GET /api/agent-runs/source-readiness, 1 call each)

- 10 ELIGIBLE: identity matched + starter enriched + market backed_depth + bookCount 9 + regime-eligible.
- 3 INELIGIBLE (starter gate): 822954 NYY@TB starter=missing; 824577 BOS@CWS starter=asymmetric
  (only home starter); 823201 COL@SF starter=missing.
- de-vigged consensus gaps from ONE the-odds-api h2h slate read (per-book devig, median across books;
  same method as the 07-05 screen). all games strictly pre-game (earliest first pitch 16:36Z vs
  screen at ~14:14Z).

## selected 8 (ranked by narrowest de-vigged gap, then books) + captured runs

| rank | gamePk | matchup | run id | lean | conf | consensus | devig gap | books | mktAgree | guard |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 823359 | ATL@PIT | 9700433e | away | 0.75 | away | 0.0262 | 9 | true | Pass |
| 2 | 823277 | AZ@SD | 9800433e | home | 0.75 | home | 0.0663 | 9 | true | Pass |
| 3 | 824816 | CHC@BAL | 9d00433e | home | 0.75 | home | 0.0789 | 9 | true | Pass |
| 4 | 823683 | CLE@MIN | 9e00433e | away | 0.75 | away | 0.0840 | 9 | true | Pass |
| 5 | 824251 | ATH@DET | a100433e | home | 0.75 | home | 0.0876 | 9 | true | Pass |
| 6 | 823846 | SEA@MIA | a200433e | away | 0.75 | away | 0.1031 | 9 | true | Pass |
| 7 | 823034 | MIL@STL | a900433e | away | 0.75 | away | 0.1038 | 9 | true | Pass |
| 8 | 822877 | LAA@TEX | aa00433e | home | 0.75 | home | 0.1184 | 9 | true | Unclear |

dropped by rank (eligible, beyond cap): 823606 KC@NYM gap 0.1407; 824494 PHI@CIN gap 0.1921.

## per-run verification (each verified on /rows BEFORE the next generation)

all 8: promptSource=registry; promptRouteKey
`mlb.pregame.analysis.starter_enriched_market_backed_depth.v2@v2::starter_enriched_market_backed_depth`;
attributionStatus=complete; fallbackReason null; exclusionReason null (rows ACTIVE -- they are the
intended prediction rows); marketConsensusSide/bookCount present; gamePk matched. run 1 artifact
spot-checked: prose names the market side ("the market consensus favors the Braves").

guard tally: **7 Pass (prose_matches_staged_consensus / MarketAligned) + 1 UnclearMarketAttribution
(822877, both_market_directions_asserted / UnclearDivergence) + 0 FAIL.** the Unclear is NOT the
hard-stop condition (stop = FailMarketAttributionMismatch); its prose consistently favors the Rangers
(= consensus = lean); the classifier likely tripped on the run-line phrase "Los Angeles Angels +1.5".
logged as guard-classifier-conservatism candidate for the Hardened-Regime Baseline Measurement v1
note (07-11). row stays ACTIVE (legitimate prediction row, not a diagnostic).

## disagreement yield (honest read, no claims)

all 8 runs marketAgreement=true -> **0 new market-opposed rows today** despite selecting the 8
narrowest-gap games. the ~1-per-6 opposed-row yield from the re-projection did NOT materialize on
day 1. disagreement ledger stays n=7/10 until v2 rows SETTLE and any future opposed rows arrive.
no improvement claims from unsettled v2 rows; v2 rows remain a SEPARATE regime era (never pooled
with v1 attribution rates; frozen baseline Pass 72 / FAIL 10 / Unclear 203).

## spend + source usage

- model: 8 gpt-4o-mini calls, estimatedTotalCost sum = **$0.00571** (cap $0.05; cost_log lines in
  agent-service output, all status ok). 0 retries, 0 diagnostic/QA runs created -> 0 exclusions needed.
- the-odds-api: 13 readiness screens + 1 h2h slate read + 8 generations = 22 calls.
- statsapi: free schedule reads.

## closeout confirmations

1. duplicate-active sweep at closeout: **0** (294 total rows, 255 active = +8 captures).
2. **no /reconcile calls today** (day 1 has no settlement pairing; none required).
3. **823281 NOT settled**; deliberate ledger remains 0/1; no checkpoint fired.
4. v2 rows separate regime era: confirmed (9 v2-route rows total = 8 active captures + excluded canary).
5. no prompt/confidence/scoring/threshold/buyer/schema/calibration changes; .env untouched; registry
   default OFF (canary env was process-scoped to the capture uvicorn process, now dead).
6. final service posture: agent-service STOPPED (:8000 down); DevCore.Api :5007 + devcore-sql left
   RUNNING for tomorrow's settlement pairing.

## next (day 2, 2026-07-10, ~10:20 ET)

settle the day-1 cohort FIRST if finals READY (check-settlement-finals.ps1 -> preflight-settlement.ps1
-> identity /reconcile with full residue; capture without settlement is not evidence), then day-2
capture per the same runbook + the no-backfill directive. day-3 (07-11): settle day-2, gate-4 readouts,
Hardened-Regime Baseline Measurement v1, cadence wrap -- authorization ENDS.
