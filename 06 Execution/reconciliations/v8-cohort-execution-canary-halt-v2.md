---
title: "v8 Cohort Execution -- Canary Halt v2 (attribution validated; retrieval-timing gate fail)"
type: "reconciliation"
date: "2026-07-03"
status: "halted-at-canary (operator go/no-go pending)"
project: "DAI"
slice: "v8 Targeted Measurement Cohort Execution (resume)"
repos:
  dai: "unchanged (96e1799)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - canary
  - retrieval
  - halt
related:
  - "02 Platform/decisions/0007-prompt-route-attribution-contract.md"
  - "06 Execution/reconciliations/v8-cohort-execution-canary-halt-v1.md"
---

# v8 Cohort Execution -- Canary Halt v2

**slice:** resume v8 under the 9-additional-call cap; attributed canary first. **outcome: HALTED again at
the canary -- attribution now WORKS end-to-end (the prior slice is validated live), but the canary game came
back `starter_missing_market_missing`, failing the operator's `backed_depth` gate. root cause = data
availability/timing, not code.** 8 model calls unspent.

## services (all restarted on HEAD 96e1799)

devcore-sql up; DevCore.Api :5007 rebuilt+restarted (attribution /rows fields); agent-service :8000
RESTARTED (critical -- the prior instance held pre-attribution code); all /health ok. pre-game recheck at
2026-07-03T20:36Z: all 9 remaining approved games Scheduled/Preview (first pitch 07-04T17:35Z onward) --
all pre-game.

## attributed canary (paid model call 2/10; 1 this session)

generated MIN@NYY (823526) -> run **c149433e-f36b-1410-8173-00373db4b724**. identity ExternalGameId 823526 ok.
**Attribution contract VALIDATED live** -- /rows shows:
- `observedDataRegime = starter_missing_market_missing`
- `selectedDataRegime = null` (routing off -- correct, not overloaded)
- `selectedPromptPath = live`, `attributionStatus = partial`, `livePromptTemplateKey = mlb.matchup.analysis.live`
This is exactly the prior slice working: a live-path run is now measurement-grade.

**BUT the regime is missing/missing, not backed_depth** -> operator gate FAILED. decision: leanSide null,
confidence 0.375 (abstention). model-call input only 2648 tokens (confirms an empty-data prompt).

## root cause (data/timing, NOT code)

DevCore.Api retrieval log for this run:
- starter: `mlb starters: NYY vs MIN on 2026-07-04 -- starters not yet announced (home=tbd, away=Zebby
  Matthews)`. the HOME (Yankees) starter reads **TBD** in the retrieval endpoint even though the schedule
  feed (v1.1 feed/live) lists Carlos Rodon -> starter context treated missing.
- market: `baseball market: no odds event matched mlb NYY vs MIN (eventId=-) on 2026-07-04` -> odds not yet
  posted/matched for a game ~21h out -> market missing. (note: the odds call is `cost=PaidExternal` -- each
  generation also consumes a the-odds-api quota call regardless of match.)
Retrieval is .NET-side (MlbStarterClient + OddsMarketClient) and was NOT touched by this slice. Canary #1
(SD@LAD be49433e, last session) DID get enriched starter + 3-book backed_depth -- so retrieval works when
the data is available; the difference is availability/timing for THIS game right now.

## verdict = STOP (operator gate failed)

canary regime != enriched_market_backed_depth -> **halted; 8 model calls unspent.** did not generate the
other 8. no improvising past the gate. cumulative v8: 2 model calls (be49433e backed_depth 0.80;
c149433e missing 0.375), both approved games, both unsettled (07-04, not final).

## go/no-go for the operator

Root cause is timing: ~21-29h before first pitch, home starters can be TBD in the retrieval endpoint and
odds are not yet posted (the plan doctrine: "enriched-starter pool fills ~24h out; market needs odds at
capture"). Options:
- **(A) RESUME ON GAME DAY (recommended).** regenerate on 2026-07-04, a few hours before each game's first
  pitch, when starters are confirmed (non-TBD) and odds are posted. re-run the attributed canary; proceed
  only for games whose /rows shows `observedDataRegime=starter_enriched_market_backed_depth`. attribution
  makes this a cheap per-run check before spending. still pre-game only; 8 model calls remain under cap.
- **(B) RETRIEVAL-ROBUSTNESS DIAGNOSTIC (separate slice).** investigate the endpoint discrepancy
  (MlbStarterClient home=tbd vs schedule=Rodon; odds no-match). candidate: fall MlbStarterClient back to the
  schedule feed's probablePitcher when the boxscore endpoint shows tbd. observability/retrieval only -- no
  prompt/decision change. do BEFORE resuming if the tbd/no-match pattern recurs on game day.
- **(C) accept a mixed cohort now** -- NOT recommended: most games this far out will abstain
  (missing/missing), wasting calls on low-value no-decision runs; the 0.80 bucket would not move.

**Recommendation: (A).** hold the 8 remaining calls; resume on 07-04 closer to first pitch. the attribution
contract now guarantees every future run is self-reporting its regime, so game-day generation can verify
backed_depth per game before committing calls.

## safety ledger

paid model calls this session **1** (c149433e; cumulative v8 = 2; cap 10, 8 unspent); paid odds-api calls
this session 1 (per-generation external, no match); new game runs 1; reconciliation writes 0 (not final);
DB migrations 0; prompt/prompt-selection/decision/confidence/buyer/denominator changes NONE; registry
routing enabled NO; no backfill; no unapproved games generated; in-progress/final games generated NONE.
