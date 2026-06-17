# Reconcile 9 Active Usable MLB Runs v1

**date:** 2026-06-16 (wait-only) / 2026-06-17 (settlement execution)
**status:** SETTLED -- all 9 target games Final, all 9 reconciled via `POST /api/agent-runs/reconcile` (SingleMatch each). Outcomes/evaluations moved 12/12 -> 21/21. 7 correct, 2 incorrect, 0 inconclusive. No code, no model calls, no generation, no migration. See the settlement-execution section below; the original wait-only record is retained as history.
**classification:** settlement execution complete; calibration evidence produced (thin-fulfilled reads grade 7/9 correct).

**Anchor:** Settlement first; calibration judgment second.

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

---

# Settlement Execution Pass (2026-06-17)

## 1. Problem statement

Execute the reconciliation deferred by the 2026-06-16 wait-only pass. The 9 active directionally-usable MLB runs (7 original active + 2 rerun active, all `LeanSide=home`) now need outcome/evaluation records so the next slice can judge whether observed-mode `PerceiveFulfillment` (`FulfilledWithThinCoverage`) has calibration value. Reconciliation only -- the existing matcher path, no analyzer/model/generation/Probe/advisory/enforcement.

## 2. Principal engineer review

1. **Still the correct 9 active usable runs?** Yes. DB confirms all 9 carry `SourceProvider=mlb_statsapi`, the expected gamePks, `LeanSide=home`, `ExclusionReason=null`.
2. **Null / superseded still excluded?** Yes. `5703433e` (824993, active null) held out; `2e03433e`/`3603433e`/`4203433e` confirmed `ExclusionReason=superseded` in DB -- so the two shared gamePks (823046, 822887) resolve to a single active run each (no `MultipleMatches`).
3. **All target games Final?** Yes. StatsAPI `abstractGameState=Final` / `detailedState=Final` for all 9.
4. **Existing reconcile path still correct?** Yes. `POST /api/agent-runs/reconcile`, identity-keyed `(TenantKey, SourceProvider, ExternalGameId)`, `SingleMatch` -> one outcome + one evaluation via the shared write path; tenant-scoped (dev bypass tenant 1, which owns the runs).
5. **Creates outcome/eval without changing analyzer?** Yes. The endpoint writes `AgentRunOutcome` + derived `AgentRunEvaluation` only; lean comes from the denormalized `AgentRun.LeanSide` column, no `OutputJson` parse, no model call.
6. **Calibration contaminants in this slice?** Reconciling against a non-final score, posting an in-progress game, or admitting a null/superseded run as an active candidate. All avoided by the settlement gate + exclusion check.
7. **Deferred after reconciliation?** Advisory mode, enforcement, live Probe execution, buyer display, threshold changes, moderate/rich real-data validation, structured Question trace.
8. **How this supports the next review?** It produces the `EvalStatus` ground truth for the 9 thin-fulfilled reads, which is the missing input for PerceiveFulfillment-vs-Outcome Calibration Review v1.

## 3. Settlement gate result

Checked at `2026-06-17` against StatsAPI (`schedule?sportId=1&gamePk=...`). **9 of 9 Final.**

| gamePk | game | state | final (away @ home) |
|---|---|---|---|
| 823452 | MIA @ PHI | Final / Final | MIA 0 @ PHI 7 |
| 822724 | KC @ WSH  | Final / Final | KC 3 @ WSH 7 |
| 824505 | NYM @ CIN | Final / Final | NYM 0 @ CIN 12 |
| 824666 | COL @ CHC | Final / Final | COL 4 @ CHC 5 |
| 824181 | DET @ HOU | Final / Final | DET 9 @ HOU 3 |
| 825071 | LAA @ AZ  | Final / Final | LAA 3 @ AZ 4 |
| 823938 | TB @ LAD  | Final / Final | TB 3 @ LAD 4 |
| 823046 | SD @ STL  | Final / Final | SD 0 @ STL 3 |
| 822887 | MIN @ TEX | Final / Final | MIN 4 @ TEX 2 |

Gate passed for all 9 -> proceed.

## 4. Reconciliation table

All posts: `SourceProvider=mlb_statsapi`, `Source=statsapi`, `ResolvedUtc=2026-06-16T06:00:00Z` (cohort settlement window), `SourceRef=statsapi schedule URL`. All returned `MatchKind=SingleMatch`.

| # | run | gamePk | teams | final (H-A) | OutcomeStatus | winner | LeanSide | eval | cohort | PF decision | SS band |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 2303433e | 823452 | Marlins @ Phillies | 7-0 | home_win | PHI | home | correct | original | FulfilledWithThinCoverage | thin |
| 2 | 2803433e | 822724 | Royals @ Nationals | 7-3 | home_win | WSH | home | correct | original | FulfilledWithThinCoverage | thin |
| 3 | 2a03433e | 824505 | Mets @ Reds | 12-0 | home_win | CIN | home | correct | original | FulfilledWithThinCoverage | thin |
| 4 | 3403433e | 824666 | Rockies @ Cubs | 5-4 | home_win | CHC | home | correct | original | FulfilledWithThinCoverage | thin |
| 5 | 3d03433e | 824181 | Tigers @ Astros | 3-9 | away_win | DET | home | incorrect | original | FulfilledWithThinCoverage | thin |
| 6 | 4103433e | 825071 | Angels @ Diamondbacks | 4-3 | home_win | AZ | home | correct | original | FulfilledWithThinCoverage | thin |
| 7 | 4903433e | 823938 | Rays @ Dodgers | 4-3 | home_win | LAD | home | correct | original | FulfilledWithThinCoverage | thin |
| 8 | 5003433e | 823046 | Padres @ Cardinals | 3-0 | home_win | STL | home | correct | rerun | FulfilledWithThinCoverage | thin |
| 9 | 5403433e | 822887 | Twins @ Rangers | 2-4 | away_win | MIN | home | incorrect | rerun | FulfilledWithThinCoverage | thin |

The settlement source provider used for all 9 is StatsAPI (`statsapi`). PF decision + SS band captured from the read-only `GET /artifact` projection (sampled 2303433e, 2a03433e, 3d03433e, 5403433e; all `enforcementMode=observed`, `decision=1`/`FulfilledWithThinCoverage`, `band=thin`, reason `thin_sport_critical_grounded`); the remaining 5 were `FulfilledWithThinCoverage`/thin in the prior observability review and unchanged.

## 5. Outcome / evaluation summary

| | outcomes | evaluations |
|---|---|---|
| before | 12 | 12 |
| after | 21 | 21 |

+9 outcomes, +9 evaluations -- exactly one per reconciled Final run. Whole-table eval breakdown after: correct 12, incorrect 5, inconclusive 4 (21 total).

## 6. Original vs rerun cohort summary

- **Original active (7):** 2303433e, 2803433e, 2a03433e, 3403433e, 3d03433e, 4103433e, 4903433e -> 6 correct, 1 incorrect (3d03433e).
- **Rerun active (2):** 5003433e (correct), 5403433e (incorrect).
- Both rerun runs matched `SingleMatch` despite sharing a gamePk with a superseded run, confirming the exclusion contract works end-to-end.

## 7. Correct / incorrect / inconclusive totals (this slice)

7 correct / 2 incorrect / 0 inconclusive. Both incorrect are home leans on away wins: 3d03433e (DET 9 @ HOU 3) and 5403433e (MIN 4 @ TEX 2).

## 8. Skipped / pending runs

None. All 9 were Final and reconciled. No identity/tenant/source/matcher skips.

## 9. Null / superseded exclusion confirmed

`5703433e` (824993 active null) not posted. Superseded `2e03433e` (823046), `3603433e` (822887), `4203433e` (824993) confirmed `ExclusionReason=superseded` and not reconciled as active candidates. Run eligibility unchanged (10 active / 3 superseded).

## 10. No analyzer behavior changed

No code change (dai clean, ahead 3). No model call (FastAPI :8000 not running; no python/uvicorn process). No sports generation. No prompt/confidence/lean/posture change. No advisory/enforcement activation. No Probe execution. No Tool Gateway call. No migration. Git diff is docs-only; the only DB mutations are the 9 outcome/evaluation rows via the existing reconcile write path.

## 11. Observed PerceiveFulfillment remained read-only

The `PerceiveFulfillment` projection is derive-on-read on `GET /artifact`, computed beside `SourceSufficiency`, persisted by nothing, consumed by no generation/matcher/evaluator/buyer surface. Post-reconcile reads still project `observed` mode `FulfilledWithThinCoverage` (decision=1) at `thin` band -- unchanged by the outcome writes.

## 12. What this means for PerceiveFulfillment-vs-Outcome Calibration Review v1

The review now has its ground truth: **all 9 directionally-usable runs projected `FulfilledWithThinCoverage` (observed, thin) and graded 7 correct / 2 incorrect = 77.8%.** This is the first real correlation sample between thin-fulfilled Perceive reads and directional correctness. It is a single thin band with no moderate/rich or `ProbeRequired`/`BlockedNotEvaluable` contrast (every active MLB run grounds exactly identity_schedule + starting_pitching), so it informs but does not by itself justify advisory promotion. n=9 is small; treat 77.8% as directional, not a calibrated threshold.

## 13. Recommended next slice

**PerceiveFulfillment-vs-Outcome Calibration Review v1** (read-only): join the 9 `EvalStatus` rows to their observed `PerceiveFulfillment` decision + `SourceSufficiency` band, state the thin-fulfilled hit rate, and decide whether observed mode has earned *advisory* authority or needs a larger / more varied (moderate/rich, ProbeRequired) sample first. No code, no model spend. Advisory/enforcement remain deferred until that review concludes.
