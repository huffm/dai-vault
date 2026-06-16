# Perceive Fulfillment Observability Review v1

**date:** 2026-06-15
**status:** review only. read-only inspection of the live observed PerceiveFulfillment projection against real MLB artifacts. no code change (no projection bug found), no model calls, no generation, no reconciliation, no outcomes/evaluations or eligibility mutated (12/12). observed mode stays live; advisory/enforcement remain deferred.
**classification:** observability review.

**Anchor:** observed mode earns trust by matching real artifacts before it earns authority.

## Problem statement

Perceive Fulfillment Observed Activation v1 made `AgentRunArtifactDto.PerceiveFulfillment` a live read-time projection over the derived `SourceSufficiency`, in `observed` mode (compute, do not enforce). Before any advisory/enforcement work, confirm on real artifacts that the projection (1) reflects actual SourceSufficiency and source provenance, (2) distinguishes source insufficiency from no directional separation, (3) does not mutate live behavior, (4) is useful enough to support later advisory/enforcement, and (5) does not overstate readiness while real MLB data is thin.

## Senior principal engineer review

1. **What would prove observed is accurate enough to keep live?** It reproduces the known cohort exactly with no false signal: directionally-usable thin runs project `FulfilledWithThinCoverage`; runs that were null because the starter was unannounced project `PrimaryFulfillmentRequired` with `primaryFulfillmentGroups=[starting_pitching]`; the grounded-but-null run projects `FulfilledWithThinCoverage` and preserves `NoDirectionalSeparation`. All 13 reviewed runs matched doctrine (table below). That is the accuracy proof.
2. **What would prove it misleading/premature?** If a thin grounded run demanded fulfillment, if `NoDirectionalSeparation` collapsed into a source-insufficiency reason, if `market_odds` became required/primary for MLB, or if `moderate`/`rich` were asserted on data that is actually thin. None observed.
3. **Specific evidence to review before advisory/enforcement:** reconciled OUTCOMES. We do not yet know whether `FulfilledWithThinCoverage` reads are actually accurate enough to act on -- the 9 active usable runs are pending settlement. Advisory authority must wait until those reconcile and we can ask "do thin-fulfilled reads grade correct often enough?".
4. **Distinction preserved?** Yes -- verified on real data. `824993` (grounded starter, null lean) -> `FulfilledWithThinCoverage` + `NoDirectionalSeparation`; `823046`/`822887` (starter unavailable) -> `PrimaryFulfillmentRequired` + `MissingCriticalGroup`. The two null kinds are cleanly separated.
5. **MLB profile strictness?** Appropriate **for observed**: requires only `starting_pitching`, minimum band `thin` (honest -- real MLB grounds one signal), `market_odds` recommended-not-required (correctly `future`/unsupported in the catalog, so never an active primary path), probe fallback disabled. Not too strict (does not demand moderate/rich), not too loose (still flags a missing starter). It is NOT yet suitable for enforcement, where thin-only would pass nearly everything.
6. **Contaminates calibration/buyer?** No. It is a read-only `GET /artifact` projection -- not persisted, not in OutputJson, not consumed by generation/matcher/evaluator, not on any buyer surface. Outcomes/evaluations (12/12) and eligibility (10 active / 3 superseded) are unchanged.
7. **Deferred even if good:** advisory mode, soft/hard enforcement, live Probe execution, structured Question trace, tenant/product profile overrides, buyer display, WNBA, moderate/rich validation on real data, persisting the projection.
8. **Overengineering to avoid:** giving the enforcement modes behavior; persisting the observation; wiring it into generation; per-tenant profile overrides; and -- most of all -- promoting to advisory on a zero-reconciled-outcome sample.

## Review cohort

10 active (7 original directionally-usable, 2 rerun directionally-usable, 1 active null) + 3 superseded historical-comparison runs.

## Per-run observation table (live projection)

All MLB; enforcementMode `observed`; required group `starting_pitching`; allowPrimaryFulfillment true; allowProbeFallback false; recommendedProbeGroups empty for all (fallback disabled).

| run | gamePk | cohort | leanSide | band | nullReason | decision | primaryFulfillmentGroups | matches doctrine |
|---|---|---|---|---|---|---|---|---|
| 2303433e | 823452 | orig active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 2803433e | 822724 | orig active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 2a03433e | 824505 | orig active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 3403433e | 824666 | orig active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 3d03433e | 824181 | orig active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 4103433e | 825071 | orig active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 4903433e | 823938 | orig active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 5003433e | 823046 | rerun active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 5403433e | 822887 | rerun active | home | thin | - | FulfilledWithThinCoverage | - | yes |
| 5703433e | 824993 | active null | null | thin | NoDirectionalSeparation | FulfilledWithThinCoverage | - | yes -- baseline met, null is no-edge |
| 2e03433e | 823046 | superseded | null | insufficient | SourceInsufficient_MissingCriticalGroup | PrimaryFulfillmentRequired | starting_pitching | yes -- starter was unavailable |
| 3603433e | 822887 | superseded | null | insufficient | SourceInsufficient_MissingCriticalGroup | PrimaryFulfillmentRequired | starting_pitching | yes -- starter was unavailable |
| 4203433e | 824993 | superseded | null | thin | NoDirectionalSeparation | FulfilledWithThinCoverage | - | yes |

(`decision` serializes as an int in the DTO: 0 Fulfilled, 1 FulfilledWithThinCoverage, 2 PrimaryFulfillmentRequired, 3 ProbeRequired, 4 BlockedNotEvaluable. A future readability nit -- a string-enum converter -- is noted, not fixed here.)

## Findings

- **Accurate projections:** all 13. The projection faithfully reflects `SourceSufficiency` (band, grounded/missing groups, null reason) and the run's source provenance (identity_schedule + starting_pitching grounded for active runs).
- **Misleading projections:** none.
- **Active vs superseded comparison:** the projection is computed per-artifact and is independent of eligibility -- superseded runs still project (useful for historical comparison) while remaining excluded from active matching. Observed PerceiveFulfillment and Run Eligibility are correctly orthogonal; the projection does not read or change `ExclusionReason`.
- **Unknowns:** (a) whether `FulfilledWithThinCoverage` predicts a *correct* read -- unknowable until the 9 active runs settle and reconcile; (b) `moderate`/`rich` and `ProbeRequired`/`BlockedNotEvaluable` decision branches are never exercised on real MLB data (every active MLB run grounds exactly one signal -> thin), so the upper bands and the probe/blocked paths are unit-tested only, not field-proven.

## Should observed computation remain live?

**Yes.** It is accurate on every reviewed run, fully read-only, non-persisted, and consumed by nothing in the generation/matcher/evaluator/buyer path. It adds a legible control surface at zero behavioral cost.

## Is advisory mode ready?

**No.** Advisory implies the projection is trustworthy enough to influence a reviewer's or the system's posture. That requires reconciled-outcome evidence that thin-fulfilled reads are accurate often enough -- which does not exist yet (the 9 usable runs are pre-settlement). Promote to advisory only after that review.

## Is enforcement ready?

**No -- well off.** Soft/hard enforcement would change whether a run proceeds. With MLB grounding a single signal, an enforced minimum-band thin would pass nearly everything (low value) and a stricter bar would block almost everything (false negatives). Enforcement needs: more grounded source groups (a real `market_odds`/supporting path), reconciled-outcome calibration that a band threshold actually separates good from bad reads, and tenant/product policy. None exist.

## What must be reviewed after reconciliation outcomes are available

Once the 9 active usable runs settle and reconcile: cross-tabulate `PerceiveFulfillment.decision` / `SufficiencyBand` against `EvalStatus` (correct/incorrect/inconclusive). The question is whether `FulfilledWithThinCoverage` reads grade meaningfully better than `PrimaryFulfillmentRequired`/`insufficient` ones. Only a real correct/incorrect signal justifies advisory.

## What remains deferred

advisory mode; soft enforcement; hard enforcement; live Probe execution; structured Question trace; tenant/product overrides; buyer display; WNBA support; moderate/rich real-data validation; persisting the projection; the string-enum serialization nit.

## Recommended next slice

Keep observed mode live, change nothing in code. The gating next step is **reconciliation of the 9 active usable MLB runs after settlement** (the calibration-evidence priority), followed by a **PerceiveFulfillment-vs-Outcome Calibration Review v1** that decides whether advisory mode is justified. Do not promote to advisory before that evidence exists.
