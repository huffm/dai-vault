# Deferred Slice Readiness Triage v1

**date:** 2026-06-16
**status:** triage / docs-control only. read-only inspection while MLB reconciliation waits for settlement. no code, no model calls, no reconciliation, no generation, no advisory/enforcement, no Probe execution. no runtime behavior changed.
**classification:** deferred-work triage.

**Anchor:** do not start deferred work until you know whether it is still deferred.

## Problem statement

While the 9 active usable MLB runs wait for their games to be Final, determine which deferred sports-v1 slices are now ready, blocked, superseded, or already absorbed -- specifically whether Probe Fallback Catalog Integration Readiness v1 is still needed or was already completed by Perceive Fulfillment Observed Activation v1.

## Senior principal engineer review

1. **Ready now?** No full slice is ready; the only ready items are this docs-status correction (mark the integration absorbed) and a cosmetic micro-nit (`decision` serializes as an int). Neither is blocked, but neither is substantive work.
2. **Blocked by outcome reconciliation?** Reconcile 9 Active Usable MLB Runs; PerceiveFulfillment-vs-Outcome Calibration Review; advisory-mode promotion -- all need the 9 reconciled `EvalStatus` rows, which do not exist yet (0/9 Final at last check).
3. **Blocked by Question trace / tenant policy / source coverage / Probe safety?** Structured Question trace (needs model output -- no deterministic producer); soft/hard enforcement (needs source coverage + calibration + tenant policy); live Probe execution (needs Tool Gateway + execution safety; dormant chain, ledger entries 1-3); tenant/product profile overrides; moderate/rich real-data validation (MLB grounds one signal -> all thin).
4. **Was Probe Fallback Catalog Integration already absorbed?** **Yes.** `PerceiveFulfillmentProjection.Build` already reads the catalog and feeds the policy; the integration slice has no remaining work.
5. **Does the projection translate catalog/source reality into policy inputs?** Yes -- `ActivePrimaryGroups` / `ActiveFallbackOrProxyGroups` resolve profile `RequiredGroups` against `ProbeFallbackCatalog.Find(...)` (IsSupported + PrimaryToolId / FallbackToolIds / ProxyToolIds) and pass the resulting `primaryFulfillmentGroups` / `probeFallbackGroups` into `PerceiveFulfillmentPolicy.Evaluate`.
6. **Mark the integration slice completed/superseded?** Yes -- absorbed by Perceive Fulfillment Observed Activation v1.
7. **Missing adapter to do now?** None -- it is done. No code this slice.
8. **Dangerous to start before the 9 MLB outcomes reconcile?** Advisory/enforcement activation, any calibration-threshold change, and promoting/trusting moderate/rich logic -- all would act on bands not yet validated against outcomes.

## Probe Fallback Catalog Integration Readiness v1 -- specific finding

The eight integration checks were each verified true in the live code (read-only):

| # | check | result | evidence |
|---|---|---|---|
| 1 | reads ProbeFallbackCatalog | yes | `SportSufficiencyProfile.cs` `ProbeFallbackCatalog.Find(group, profile.Competition)` |
| 2 | uses sport/profile requirements | yes | `SportSufficiencyProfiles.Find` -> `RequiredGroups` (MLB `[starting_pitching]`) |
| 3 | determines supported primary/fallback/proxy paths | yes | `ActivePrimaryGroups` (IsSupported + PrimaryToolId); `ActiveFallbackOrProxyGroups` (FallbackToolIds/ProxyToolIds) |
| 4 | feeds primary/fallback groups into the policy | yes | `PerceiveFulfillmentPolicy.Evaluate(competition, sufficiency, primaryGroups, probeGroups)` |
| 5 | projects read-only in `AgentRunArtifactDto.PerceiveFulfillment` | yes | `AgentRunsController.GetArtifact` -> `PerceiveFulfillmentProjection.Build` -> DTO field |
| 6 | does not execute Probe | yes | catalog is metadata ("does not execute probe"); projection path has no executor |
| 7 | does not call Tool Gateway | yes | no `ToolGateway`/`Execute`/`HttpClient`/`ProbeRefreshExecutor` in the projection path |
| 8 | does not mutate analyzer behavior | yes | read-only GET projection; non-mutation verified in Perceive Fulfillment Observability Review v1 (outcomes/evals 12/12, eligibility unchanged) |

All true -> **the integration slice is effectively absorbed; do not add code.**

## Deferred slice candidate status table

| candidate | status | reason |
|---|---|---|
| Probe Fallback Catalog Integration Readiness v1 | **superseded / absorbed** | implemented inside Perceive Fulfillment Observed Activation v1 (8/8 integration checks true) |
| Reconcile 9 Active Usable MLB Runs v1 | blocked by reconciliation | 0/9 games Final at last check; pure timing |
| PerceiveFulfillment-vs-Outcome Calibration Review v1 | blocked by reconciliation | needs the 9 reconciled `EvalStatus` rows |
| Advisory-mode promotion | blocked by reconciliation | needs outcome evidence that thin-fulfilled grades correct |
| moderate/rich band real-data validation | blocked by source support | MLB grounds one signal -> every active run is thin |
| Structured Question trace / decision-uncertainty producer | blocked by Question trace | requires model output; no deterministic producer exists |
| Live Probe execution / fallback | keep deferred | Tool Gateway + execution safety; dormant chain (entries 1-3) |
| Soft / hard enforcement | keep deferred | needs source coverage + calibration + tenant policy |
| Tenant / product profile overrides | keep deferred | no tenant policy layer yet |
| Buyer-facing fulfillment display | keep deferred | observed control is internal; not buyer-validated |
| WNBA support | keep deferred | no catalog/seed/odds/analyzer path (entry 26) |
| `decision` int->string-enum serialization | ready now (cosmetic) | optional future micro-slice; not started here (no-code boundary) |

## Recommended next action while waiting for settlement

Triage/docs only (this slice). Do not start advisory, calibration, enforcement, Question-trace, or Probe-execution work -- all are blocked or dangerous pre-reconciliation. The cosmetic string-enum nit could be a tiny safe micro-slice later but is out of this slice's no-code boundary.

## Recommended next action after reconciliation

1. Rerun Reconcile 9 Active Usable MLB Runs v1 after all 9 are Final (~`2026-06-16 05:30Z`) via the proven statsapi path.
2. Then PerceiveFulfillment-vs-Outcome Calibration Review v1 -- cross-tabulate `PerceiveFulfillment.decision` / `SufficiencyBand` against `EvalStatus` to decide whether advisory mode is justified.

## No runtime behavior changed

Read-only triage. No code, no DTO/projection change, no model call, no reconciliation, no generation, no advisory/enforcement, no Probe execution, no eligibility/outcome mutation.
