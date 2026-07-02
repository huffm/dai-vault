---
title: "Outcome Reconciliation Follow-up v7"
type: "reconciliation"
date: "2026-07-02"
status: "complete-partial"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v7"
repos:
  dai: "unchanged (6c13b1d)"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - outcome
  - calibration
related:
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v6.md"
  - "06 Execution/plans/live-calibration-cohort-planning-v1.md"
  - "04 Products/sports-v1/calibration/calibration-result-review-v1.md"
---

# Outcome Reconciliation Follow-up v7 (partial: 1 of 9 settled, 8 time-deferred)

**slice:** settle the nine 07-02 backlog games when Final; pooled reassessment across settled slates
**status:** partial complete 2026-07-02 ~16:40 ET (1 Final settled; 2 live + 6 pre-game deferred to v7b)
**repos:** `dai` unchanged (`6c13b1d`, post boundary-hardening arc). `dai-vault` docs-only (this doc + slice entry).

## start state

dai clean/synced `6c13b1d` (0/0); dai-vault clean/synced `71bac25`+arc docs (0/0); synopsis excluded as always.
services cold-started for this pass: docker engine + `devcore-sql` started, platform api built from HEAD
`6c13b1d` and run with `ASPNETCORE_ENVIRONMENT=Development` (`/health` ok). note: this is the first settlement
pass through the new auth boundary -- unauthenticated local calls passed via the explicit dev bypass
(`Development` + `Dev:EnableBypassAuth`), which is exactly the intended ops posture.

## probe (free statsapi schedule, 9 gamePks, ~20:33Z)

| gamePk | game | status | score |
|---|---|---|---|
| 823442 | PIT@PHI | **Final** | **6-1 away** |
| 823765 | CIN@MIL | Live (In Progress) | 7-2 |
| 824335 | MIA@COL | Live (In Progress) | 4-3 |
| 824416 | CWS@CLE | Pre-Game | 22:40Z |
| 824906 | STL@ATL | Pre-Game | 23:15Z |
| 824093 | TB@KC  | Pre-Game | 23:40Z |
| 822884 | DET@TEX | Pre-Game | 00:05Z |
| 823119 | LAA@SEA | Scheduled | 01:40Z |
| 823935 | SD@LAD | Scheduled | 02:10Z |

partition: final/eligible 1; not final 8 (2 live, 6 pre-game); identity-unsafe 0; missing source 0;
already reconciled 0; excluded 0. `/rows` confirmed all nine active (`exclusionReason` null -- the new
per-row field from exclusion visibility v1 served this read), one run per gamePk, `sourceProvider=mlb_statsapi`.

## reconcile (1 write)

precheck `GET /reconcile-precheck?sourceProvider=mlb_statsapi&externalGameId=823442`: 1 active run,
SingleMatch, "Identity POST /reconcile is safe."

`POST /reconcile` `{mlb_statsapi, 823442, away_win, home 1, away 6, source statsapi_final}` ->
**SingleMatch, evaluatedRunId f940433e..., evalStatus incorrect** (run f940433e, route
`starter_enriched_market_missing`, lean home conf 0.675; Phillies home lost 1-6).
direction-integrity guard untriggered (consistent artifact); post-write precheck: unreconciledActive 0,
hasOutcome true (idempotent state confirmed read-only; no second write attempted).

**this is the first directional settlement ever on the `enriched_market_missing` route: 0/1, and it is
another "leaned home, away won" miss -- the dominant failure mode continues onto a second route.**

## metrics delta (before -> after)

totalRows 263 -> 263; reconciledRows 84 -> 85; noDecisionRows 11 -> 11; matched 51 -> 51;
unmatched 33 -> 34; **matchRate 0.6071 -> 0.6000** (51/85). total outcomes 95 -> 96.

## pooled reassessment (read-only tool, full corpus)

`pooled_calibration_report.py --url .../rows`: reconciled 96 (85 directional + 11 no-decision),
**10 settled slates** (slate = createdUtc capture date; 823442 pools under its 06-30 capture slate).

gates: slates 10/3 MET; enriched_market_missing directional 1/1 MET (barely -- n=1);
market disagreement 4/2 MET; confidence buckets n>=15 **NOT MET** (5 of 6 buckets under 15;
only 0.75 qualifies at n=61). **conclusionsAllowed: FALSE.**

descriptive (not actionable, per gates):
- overall directional accuracy 51/85 = **0.600**.
- confidence remains non-predictive and non-monotonic: 0.75 bucket (n=61) 0.607; **0.80 bucket (n=12)
  0.500** -- the highest-confidence band is still a coin flip; 0.70 (n=6) 0.833 on tiny n.
- home bias persists: home leans (n=61) 0.574 vs away leans (n=24) 0.667.
- market agreement (n=45 with a market side): agree 0.634 (n=41); disagree 0.500 (n=4, far too small).
- routes: enriched_backed_depth 0.625 (n=16); enriched_market_missing 0.0 (n=1); legacy/unknown 0.603 (n=68).
- slate variance is wild (0.25 to 1.00) at these per-slate sizes -- expected noise.

## verdict

**no tuning. continue measurement.** the tool's own gates refuse conclusions (confidence-bucket
thinness), and every prior directional finding is unchanged in direction: ~0.60 overall (inside noise
vs home base rate), confidence non-predictive with the top band inverted, home bias persistent.
the one new signal -- enriched_market_missing 0/1 -- is a single game and proves nothing except that
the route now has its first datapoint. settling the remaining 8 (v7b, tonight's finals) adds up to
2 more directional (823765, 823119, both home leans) + 6 no-decision.

## next action

**v7b:** re-probe after ~2026-07-03T05:00Z; settle finals per this doc's partition (all currently
SingleMatch-safe; re-run precheck before each write anyway); then re-run the pooled report. no paid
calls, no new runs, no denominator changes.

## safety ledger

paid calls 0; new game runs 0; reconciliation writes 1 (final + identity-safe only);
migrations/schema 0; prompt/decision/buyer changes none; /metrics denominator untouched;
auth/tenant boundaries untouched. no dai code changed -> no tests needed (tooling behaved
exactly as specified; 1035/1035 from the tenant-scoping slice remains the current code baseline).
