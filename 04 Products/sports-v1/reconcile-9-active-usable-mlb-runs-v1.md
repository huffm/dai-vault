# Reconcile 9 Active Usable MLB Runs v1

**date:** 2026-06-16
**status:** wait-only -- blocked on settlement. at execution time 0 of the 9 target games are Final (7 in progress, 2 pre-game), so no `POST /api/agent-runs/reconcile` was submitted, no outcome was fabricated, no in-progress score was posted, and no outcome/evaluation rows were written (12/12 unchanged). no code, no model calls, no generation.
**classification:** pre-settlement wait-only; no calibration evidence produced.

**Anchor:** outcome evidence is the next control signal.

## Problem statement

Capture outcome/evaluation records for the 9 active directionally usable MLB runs (Stage 0 observed cohort) so the next slice can review whether `FulfilledWithThinCoverage` correlates with correct decisions. Reconciliation uses the existing matcher path only.

## Senior principal engineer review

1. **Correct cohort?** Yes -- 7 original active + 2 rerun active, all `LeanSide=home`, all active (`ExclusionReason` null), confirmed in the prior slices and the eligibility check (10 active total, of which 5703433e is the active null held out).
2. **Superseded/null excluded?** Yes -- `5703433e` (active null, `824993`) and `2e03433e`/`3603433e`/`4203433e` (superseded) are not in the 9.
3. **All target games settled?** **No -- 0 of 9 Final.** This is the blocker (gate result below).
4. **Correct reconciliation path?** Yes -- `POST /api/agent-runs/reconcile` against StatsAPI finals, the proven path from the first reconciliation slice.
5. **Produces outcome/eval without changing analyzer?** Yes when run (matcher SingleMatch -> one outcome + one evaluation via the shared write path); not run this slice.
6. **What would contaminate calibration here?** Reconciling an in-progress game against a non-final score, or including a null/superseded run as an active candidate. Both avoided by the settlement gate.
7. **Deferred after reconciliation:** advisory/enforcement, and the PerceiveFulfillment-vs-Outcome calibration review (which needs these outcomes first).

## Cohort definition

| # | run | gamePk | teams | cohort |
|---|---|---|---|---|
| 1 | 2303433e | 823452 | Marlins @ Phillies | original active |
| 2 | 2803433e | 822724 | Royals @ Nationals | original active |
| 3 | 2a03433e | 824505 | Mets @ Reds | original active |
| 4 | 3403433e | 824666 | Rockies @ Cubs | original active |
| 5 | 3d03433e | 824181 | Tigers @ Astros | original active |
| 6 | 4103433e | 825071 | Angels @ Diamondbacks | original active |
| 7 | 4903433e | 823938 | Rays @ Dodgers | original active |
| 8 | 5003433e | 823046 | Padres @ Cardinals | rerun active |
| 9 | 5403433e | 822887 | Twins @ Rangers | rerun active |

Held out (not reconciled as active calibration candidates): `5703433e` (824993, active null), `2e03433e`/`3603433e`/`4203433e` (superseded).

## Settlement gate result

Current UTC at execution: `2026-06-16 00:27Z`. StatsAPI `abstractGameState` for each target gamePk:

| gamePk | game | state | partial score (home-away) |
|---|---|---|---|
| 823452 | MIA @ PHI | Live / In Progress | 5-0 (not final) |
| 822724 | KC @ WSH | Live / In Progress | 7-3 (not final) |
| 824505 | NYM @ CIN | Live / In Progress | 9-0 (not final) |
| 824666 | COL @ CHC | Live / In Progress | 1-0 (not final) |
| 824181 | DET @ HOU | Live / In Progress | 0-1 (not final) |
| 825071 | LAA @ AZ | Preview / Pre-Game | not started |
| 823938 | TB @ LAD | Preview / Pre-Game | not started |
| 823046 | SD @ STL | Live / In Progress | 0-0 (not final) |
| 822887 | MIN @ TEX | Live / In Progress | 0-3 (not final) |

**0 of 9 Final** -> wait-only. Partial scores are recorded above only as evidence the games are mid-flight; they are NOT outcomes and were NOT posted.

## Reconciliation table

None. No reconcile request was submitted for any run (no game settled).

## Outcome / evaluation summary

| | outcomes | evaluations |
|---|---|---|
| before | 12 | 12 |
| after | 12 | 12 |

No row written. Correct / incorrect / inconclusive added this slice: 0 / 0 / 0.

## Original vs rerun cohort summary

Both sub-cohorts (7 original, 2 rerun) are fully pending settlement; neither produced any reconciliation this slice.

## Skipped / pending runs

All 9 are pending settlement (7 in progress, 2 pre-game). None skipped for an identity/tenant/source reason -- timing only. The 2 pre-game runs (825071, 823938) will settle latest; the latest scheduled start is 02:10Z, so all 9 should be Final by roughly `2026-06-16 05:30Z` (allowing ~3h per nine-inning game).

## Exclusions confirmed

`5703433e` (active null) and the three superseded runs were not considered for active reconciliation. Run Eligibility (10 active / 3 superseded) is unchanged; no eligibility mutation occurred.

## No analyzer / projection change

No model call, no generation, no prompt/confidence/lean/posture change, no advisory/enforcement activation, no Probe execution, no Tool Gateway call. The observed PerceiveFulfillment projection remains read-only and unchanged (not touched this slice).

## What this means for the next PerceiveFulfillment-vs-Outcome review

Nothing yet -- that review needs reconciled `EvalStatus` for these 9 runs. It cannot start until reconciliation runs post-settlement.

## Recommended next slice

Rerun this exact reconciliation after settlement (~`2026-06-16 05:30Z`, once all 9 are Final): reconcile the 9 `(mlb_statsapi, gamePk)` keys via `POST /api/agent-runs/reconcile` with StatsAPI finals, expecting `SingleMatch` + correct/incorrect each, count +9, and the 409 guard on a re-post. Then PerceiveFulfillment-vs-Outcome Calibration Review v1 to decide whether advisory mode is justified. No code, no model spend.
