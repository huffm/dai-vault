---
title: "Outcome Reconciliation Follow-up v7b"
type: "reconciliation"
date: "2026-07-02"
status: "complete-partial"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v7b"
repos:
  dai: "unchanged (6c13b1d)"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - outcome
  - calibration
related:
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v7.md"
  - "06 Execution/plans/live-calibration-cohort-planning-v1.md"
---

# Outcome Reconciliation Follow-up v7b (partial: 1 more settled, 7 time-deferred)

**slice:** settle the remaining eight 07-02 backlog games when Final; rerun pooled reassessment
**status:** partial complete 2026-07-02 ~17:35 ET, closed on operator instruction (1 settled;
1 live + 6 pre-game deferred to a v7c pass after ~2026-07-03T05:30Z)
**repos:** `dai` unchanged (`6c13b1d`). `dai-vault` docs-only (this doc + slice entry).

## state

dai clean/synced `6c13b1d`; dai-vault ahead-1 (v7 docs commit `659e04c`, unpushed); services still up
from v7 at HEAD (api `/health` ok, devcore-sql up). backlog recheck via `/rows`: 823442 reconciled (v7),
remaining 8 active/unreconciled, one run per gamePk, all `exclusionReason` null.

## probe (statsapi, ~21:31Z)

**Final: 823765 CIN@MIL 7-2 away.** live: 824335 MIA@COL (12-4 COL, in progress). pre-game/scheduled:
824416, 824906, 824093, 822884, 823119, 823935. a background poller had been armed to wake on all-final
(deadline 06:30Z); the operator closed the slice early, so the poller was stopped after this settlement.

partition: final/eligible 1; not final 7; identity-unsafe 0; missing source 0; already reconciled 1
(823442, v7); excluded 0.

## reconcile (1 write)

precheck 823765: 1 active run, SingleMatch, identity /reconcile safe.
`POST /reconcile` `{mlb_statsapi, 823765, away_win, home 2, away 7, statsapi_final}` ->
**SingleMatch, evaluatedRunId fc40433e..., evalStatus incorrect** (lean home conf 0.675, route
`starter_enriched_market_missing`; Brewers home lost 2-7). post-write precheck: unreconciledActive 0,
hasOutcome true. direction-integrity guard untriggered.

**the enriched_market_missing route is now 0/2 -- both directional settlements are
leaned-home-away-won misses.** n=2 proves nothing statistically, but the failure mode is identical to
the corpus-wide home-bias pattern, now on a second route.

## metrics delta (v7 -> v7b)

reconciledRows 85 -> 86; unmatched 34 -> 35; matched 51 (unchanged); **matchRate 0.6000 -> 0.5930**
(51/86); noDecision 11 (unchanged); total outcomes 96 -> 97.

## pooled reassessment (rerun, full corpus)

97 reconciled / 86 directional / 11 no-decision / 10 slates (slate = createdUtc capture date).
gates: slates 10/3 MET; enriched_market_missing n=2 MET; market-disagreement 4/2 MET; confidence
buckets n>=15 **NOT MET** (0.63 n=1, 0.68 n=4, 0.70 n=6, 0.72 n=2, 0.80 n=12; only 0.75 qualifies
at n=61). **conclusionsAllowed: FALSE** -- unchanged from v7.

descriptive (not actionable): overall 51/86 = **0.593**; 0.75 bucket 0.607, **0.80 bucket 0.500**
(top band still coin flip), 0.68 bucket 0.25 (n=4, noise); home leans 0.5645 (n=62) vs away 0.6667
(n=24) -- bias slightly worse; market-agree 0.634 (n=41) vs disagree 0.500 (n=4).

## verdict

**no tuning. continue measurement.** gates closed on the same criterion as v7 (confidence-bucket
thinness). the only movement is two more identical-direction datapoints for the existing home-bias
story. n=2 on enriched_market_missing suggests nothing beyond "watch this route" -- 823119 LAA@SEA
(the third home-lean on this route) settles tonight and brings it to n=3.

## next action

**v7c after ~2026-07-03T05:30Z:** settle the remaining 7 (expected +1 directional 823119 home-lean +
6 no-decision), re-precheck each before writing, rerun the pooled report. no paid calls, no new runs,
no denominator changes.

## safety ledger

paid calls 0; new game runs 0; reconciliation writes 1 (final + identity-safe only); migrations 0;
prompt/decision/buyer changes none; /metrics denominator untouched; no dai code changed -> no tests
needed (tooling per spec; code baseline 1035/1035 unchanged).
