---
title: "Backed-Depth Dangling Row Cleanup (822882) v1"
type: "reconciliation"
date: "2026-07-05"
status: "complete -- 822882 settled no-decision (inconclusive); 0 dangling backed_depth rows"
project: "DAI"
slice: "Backed-Depth Cleanup + Divergence Capture Prep -- Phase 1"
repos:
  dai: "unchanged (32180df) -- no code edits"
  dai-vault: "docs-only (this report)"
tags:
  - reconciliation
  - calibration
  - cohort
  - registry
  - backed-depth
related:
  - "06 Execution/reports/reconciliation-last-cohort-2026-07-05-v1.md"
  - "06 Execution/reconciliations/paid-registry-routing-canary-v1.md"
  - "06 Execution/reconciliations/registry-routed-v8-backed-depth-cohort-v1.md"
---

# Backed-Depth Dangling Row Cleanup (822882) v1

**scope:** settle-or-exclude the one remaining unreconciled backed_depth registry row (822882 DET@TEX, the prior
no-lean registry canary), so the backed_depth registry route has zero dangling rows. Measurement-integrity only:
no new analyses, no new runs, no paid calls, no code/prompt/routing/confidence/buyer/schema change.

## target row (proven, not assumed)

| field | value |
|---|---|
| gamePk / externalGameId | 822882 |
| run | c849433e (c849433e-f36b-1410-8173-00373db4b724) |
| sourceProvider | mlb_statsapi |
| homeTeamRef / awayTeamRef | texas-rangers / detroit-tigers |
| leanSide | NULL (no-decision) |
| exclusionReason | NULL (active) |
| promptSource / regime | registry / starter_enriched_market_backed_depth |
| pre-state | hasOutcome=0, hasEval=0 (unreconciled) |

## classification: SafeToSettleNoDecision

- **official source (StatsAPI 822882):** `abstractGameState=Final`, `detailedState=Final`, `statusCode=F`.
  home Texas Rangers 0, away Detroit Tigers 3 -> **away_win**. DB home/away refs match StatsAPI home/away exactly
  (no inversion, no name-only match).
- **precheck** `GET /reconcile-precheck?sourceProvider=mlb_statsapi&externalGameId=822882` -> advice=IdentitySafe
  (1 active run, 1 unreconciled, backlog run c849433e). Identity `POST /reconcile` safe.
- **no-decision path supported:** the run has no lean, so the existing evaluator (`RunEvaluator.Evaluate`) returns
  `inconclusive` when leanSide is null -- the app path records the real outcome without fabricating a directional
  verdict. This is the same path used for prior no-lean runs (e.g. 28bd433e, the six 07-02 starter_missing rows).

## write performed

`POST /api/agent-runs/reconcile` (SingleMatch), Residue Contract v1 complete:

```
sourceProvider = mlb_statsapi
externalGameId = 822882
outcomeStatus  = away_win        (real result; NOT a DAI lean)
homeScore = 0, awayScore = 3
source    = statsapi_final
sourceRef = "gamePk 822882"
notes     = "Registry Routing Canary v1 (no-lean / no-decision); DET 3 @ TEX 0; via reconcile"
```

Persisted (verified in DB): `OutcomeStatus=away_win`, `HomeScore=0`, `AwayScore=3`, `EvalStatus=inconclusive`,
`LeanSide=NULL`, `WinningSide=away` (records game reality), residue source/sourceRef/notes all non-null.

**Semantic proof (no fabricated verdict):** `EvalStatus=inconclusive` -> this row does NOT count as DAI correct
or incorrect and does NOT affect the DAI hit rate. `WinningSide=away` is the game's actual winner, not a DAI
verdict; the derived evaluation is inconclusive because the run had no directional lean.

## route effect (read-only, post-write)

```
starter_enriched_market_backed_depth registry route:
  totalRows 23 | directional reconciled 22 (matched 15 / unmatched 7, matchRate 0.682) | no-decision 1 (822882) | dangling/unreconciled 0
global: noDecisionRows 17 -> 18 ; reconciledRows 94 (directional, unchanged) ; matchRate 0.6064
```

The DAI hit rate and confidence-bucket reads from `reconciliation-last-cohort-2026-07-05-v1.md` are **unchanged**
(822882 is no-decision, not directional). `conclusionsAllowed` remains FALSE.

## validation

- AgentRuns count = 273 before and after (0 new runs).
- 0 paid model calls (agent-service not started); StatsAPI (free) + devcore-sql + DevCore.Api only.
- 1 DB outcome + 1 DB evaluation written (822882 only); no other run touched.
- No code/config/prompt/routing/confidence/buyer/migration/schema change (`dai` tree = pre-existing csproj phantom only).
- 0 dangling backed_depth registry rows remain.

## outcome

**822882 SETTLED as no-decision (inconclusive).** Not excluded -- the app path safely supports no-decision
settlement, so recording the real outcome (inconclusive eval) is more accurate than an exclusion. The backed_depth
registry route now has no unreconciled rows.
