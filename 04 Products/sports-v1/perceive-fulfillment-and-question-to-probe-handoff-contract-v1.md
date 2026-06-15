# Perceive Fulfillment and Question-to-Probe Handoff Contract v1

**date:** 2026-06-15
**status:** implemented as a dormant backend contract plus docs. no runtime consumer, no gate activation, no DTO expansion, no persistence, no migration, no prompt/model/tool change, no generation, no reconciliation.
**classification:** Perceive fulfillment seam + Question/Probe boundary doctrine.

## Problem statement

Source Group Taxonomy v1 made source coverage legible through `SourceSufficiency`, but the next boundary was still implicit: when has Perceive fulfilled baseline evidence, when should primary source fulfillment be attempted, and when should a later Question/Probe path repair a targeted gap?

The slice needed to make this boundary implementation-ready without activating behavior. The doctrine is:

- Perceive fulfills.
- Question focuses.
- Probe repairs.

## Principal engineer review

1. **Why not duplicate ProbeRequest?** `ProbeRequest` already exists as the structured Interrogate/Question-to-Probe handoff: signal key, reason, nullable orchestrator-resolved tool id, priority, and confidence effect. Duplicating it would create parallel probe semantics and drift from the established doctrine that Probe may recommend investigation but does not retrieve.
2. **Why only add Perceive fulfillment now?** The missing deterministic seam was before Question/Probe: a pure read of `SourceSufficiency` that says whether Perceive fulfilled baseline evidence or whether a gap remains. That is the smallest useful contract because it consumes existing taxonomy output and does not require orchestration.
3. **Why Question stays docs-only?** A decision-focused Question layer needs model output or richer protocol trace to identify decision-relevant uncertainty honestly. There is no safe deterministic producer today, so adding `QuestionRequired` code would fake intelligence the system does not have.
4. **Why dormant-safe?** `PerceiveFulfillmentPolicy` is a static pure policy. It is not DI-registered, not called from generation, not projected on DTOs, not persisted, and not connected to Probe or Tool Gateway.
5. **What is dangerous to activate too early?** Gating live analysis, calling Probe tools, resolving fallback tools without approved source policy, treating `NoDirectionalSeparation` as source insufficiency, or using thin MLB artifacts to calibrate broad fulfillment thresholds.
6. **What overengineering was avoided?** No new probe handoff object, no Question simulator, no orchestrator, no fallback catalog, no endpoint, no migration, no persisted state, no frontend surface, and no prompt changes.

## Minimal contract chosen

Code added:

- `platform/dotnet/DevCore.Api/AgentRuns/PerceiveFulfillmentPolicy.cs`
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/PerceiveFulfillmentPolicyTests.cs`

The contract is:

- `PerceiveFulfillmentDecision`
- `PerceiveFulfillmentResult`
- `PerceiveFulfillmentPolicy.Evaluate(...)`

`Evaluate` consumes:

- competition code, used only to derive the sport-critical source group through `SourceSignalTaxonomy.SportCriticalGroup`.
- existing `SourceSufficiency`.
- explicit `primaryFulfillmentGroups`.
- explicit `probeFallbackGroups`.

Source path availability is explicit input. The policy does not infer approved tools, call catalogs, or consult runtime services.

## Why ProbeRequest was reused, not duplicated

`ProbeRequest` remains the singular structured Probe handoff. It already carries:

- `SignalKey`
- `Reason`
- `SuggestedToolId`
- `Priority`
- `ConfidenceEffect`

The new Perceive fulfillment contract may decide that a later Probe repair is required, but it only recommends source groups. It does not create probe signals, resolve tool ids, mutate `ProbeRequest`, or execute the existing dormant probe-refresh chain.

## Why Question is docs-only for now

Question should focus decision-relevant uncertainty after Perceive has fulfilled baseline evidence. Today the system has source sufficiency structure and prose protocol traces, but no deterministic Question producer that can honestly decide what uncertainty matters.

Future Question contract expectations:

- consume a structured decision trace, not prose scraping.
- identify uncertainty that affects the decision, not just missing sources.
- output a narrow focus object that can feed the existing `ProbeRequest` path when repair is warranted.
- remain separate from retrieval and tool execution.

This slice intentionally does not add `QuestionRequired` to code.

## Perceive fulfillment decision vocabulary

- `Fulfilled`
- `FulfilledWithThinCoverage`
- `PrimaryFulfillmentRequired`
- `ProbeRequired`
- `BlockedNotEvaluable`

No `QuestionRequired` value was added in code.

## Dormant policy rules

- `moderate` or `rich` sufficiency -> `Fulfilled`.
- `thin` sufficiency with the sport-critical group grounded -> `FulfilledWithThinCoverage`.
- missing sport-critical coverage plus a supported primary fulfillment group -> `PrimaryFulfillmentRequired`.
- missing sport-critical coverage plus fallback/proxy groups only -> `ProbeRequired`.
- missing sport-critical coverage plus no supported path -> `BlockedNotEvaluable`.
- `NoDirectionalSeparation` remains distinct from source insufficiency. Thin coverage with sport-critical grounded still fulfills baseline Perceive coverage.
- `ProbeRequired` recommends source groups only. It never executes Probe and never resolves a tool id.

## How this uses SourceSufficiency

`SourceSufficiency` remains the input read model:

- `Band` drives fulfilled vs thin vs insufficient decisions.
- `Groups` drives grounded critical/supporting/contextual groups.
- `Groups` plus sport-critical derivation drives missing sport-critical groups.
- `NullReason` is carried through unchanged so `NoDirectionalSeparation` is not converted into source insufficiency.

The new result exposes:

- `Decision`
- `SufficiencyBand`
- `MissingCriticalGroups`
- `MissingSportCriticalGroups`
- `GroundedCriticalGroups`
- `GroundedSupportingGroups`
- `GroundedContextualGroups`
- `NullReason`
- `PrimaryFulfillmentGroups`
- `RecommendedProbeGroups`
- `Reason`

## Boundary separation

Perceive fulfills baseline evidence. It answers: is source coverage enough to proceed, does primary fulfillment remain, is only a fallback/probe path available, or is the run not evaluable?

Question focuses decision-relevant uncertainty. It is not implemented deterministically yet because that requires model output or richer structured trace.

Probe repairs targeted gaps under approved source/tool policy. The existing `ProbeRequest` and dormant probe-refresh contracts remain the Probe-side handoff and orchestration shapes.

## Activation prerequisites

- more primary source group coverage.
- approved source/tool policy.
- Probe fallback catalog.
- structured Question trace.
- real-data validation beyond thin MLB artifacts.
- calibration impact review.

## Overengineering avoided

- no duplicate `ProbeRequest`.
- no parallel ProbeRequest-like type.
- no deterministic Question simulation.
- no runtime gate.
- no Tool Gateway call.
- no endpoint or DTO expansion.
- no persistence or migration.
- no prompt change.
- no buyer-facing change.

## Verification

- focused `PerceiveFulfillmentPolicyTests`: 9/9 passed.
- focused `SourceSignalTaxonomyTests`: 15/15 passed.
- full `DevCore.Api.Tests`: 689/689 passed.

No model calls, no generation, no reconciliation, no migrations, and no outcome/evaluation changes.

## Recommended next slice

**Probe Fallback Catalog v1** after a primary source policy review. Define which source groups may use fallback/proxy repair, with approved tool/source policy, still without executing Probe. A separate later slice should add structured Question trace only when the analyzer produces enough typed decision uncertainty to support it honestly.
