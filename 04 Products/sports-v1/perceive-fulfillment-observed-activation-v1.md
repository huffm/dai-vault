# Perceive Fulfillment Observed Activation v1

**date:** 2026-06-15
**status:** implemented as live observed read/projection metadata. no enforcement, no probe execution, no tool gateway call, no model call, no sports generation, no reconciliation, no prompt change, no confidence/posture/lean/buyer change, no persistence, no migration.
**classification:** observed control surface.

## Problem statement

Source Group Taxonomy v1 made `SourceSufficiency` derive-on-read. Perceive Fulfillment and Question-to-Probe Handoff Contract v1 added a pure `PerceiveFulfillmentPolicy`. Probe Fallback Catalog v1 added the approved source/tool option menu.

The missing step was observability: artifact reads should expose whether Perceive fulfilled baseline evidence, while preserving current analyzer behavior. This slice moves fulfillment from dormant code to live observed computation only.

Core rule: live computation, no live enforcement.

## Principal engineer review

1. **Why observed computation now:** the required inputs are deterministic and already available at artifact read time: `SourceSufficiency`, sport profile requirements, and `ProbeFallbackCatalog` source-path knowledge. This is the right point to make the future gate observable before it controls anything.
2. **Why enforcement is premature:** current MLB artifacts often max out at thin coverage; `Question` has no structured trace; the catalog is incomplete; and calibration evidence is still too small. Hard requirements now would strangle the pipeline and create false confidence in the gate.
3. **Where the computation belongs:** read/projection. Generation would risk changing behavior; persistence would freeze a value that is still a policy projection. The DTO can derive it beside `SourceSufficiency`.
4. **Why calibration is not contaminated:** the result is not written to `OutputJson`, not used by `SportsEvaluator`, not used by reconciliation, and does not mutate `LeanSide`, confidence, posture, status, outcome, evaluation, or calibration eligibility.
5. **How sport requirements stay niche-specific:** `SportSufficiencyProfile` lives in the sports/agent-run area and names sport/source-group expectations. The generic fulfillment policy remains reusable and source-agnostic.
6. **Tenant/product policy complexity avoided:** no tenant overrides, buyer display, pricing, rate-limit policy, or per-product enforcement mode was introduced.
7. **No duplicate policy objects:** this does not duplicate `ProbeRequest`, `ProbeFallbackCatalog`, `ToolRegistry`, or `ProtocolToolAccessPolicy`. It reads catalog metadata and returns diagnostic fulfillment metadata only.
8. **Dangerous early activation:** advisory or hard gating, Probe/tool execution, treating future-candidate catalog entries as supported, making market odds required for MLB, or using this value to alter confidence/posture/lean.
9. **Overengineering avoided:** no endpoint command, no database column, no migration, no frontend card, no tenant policy framework, no Question simulator, no orchestrator, and no Probe execution adapter.

## Doctrine alignment

- **Platform control seam:** the platform read path now exposes a control signal that future orchestration can observe.
- **Niche-specific profiles:** the sports niche defines source-group requirements. Generic platform code does not decide what MLB evidence requires.
- **Frontend remains packaging:** no frontend or buyer copy owns sufficiency logic.
- **Observed control before enforcement:** the future gate is visible before it can block.

Doctrine remains:

- Perceive fulfills.
- Question focuses.
- Probe repairs.

## Existing projection and policy audit

1. `SourceSufficiency` is projected in `AgentRunsController.GetArtifact` from existing artifact signals and run identity.
2. `PerceiveFulfillmentPolicy` can consume that `SourceSufficiency` directly.
3. The policy needs two path lists: primary fulfillment groups and probe fallback groups.
4. Those lists can be derived from `ProbeFallbackCatalog` without executing tools.
5. Sport profiles belong in the sports/agent-run layer, not the frontend and not generic platform core.
6. `AgentRunArtifactDto` is the right internal inspection surface.
7. The buyer-facing `AgentRunResultDto` remains unchanged.
8. Analyzer behavior is unchanged.
9. Persisted `OutputJson` is unchanged.
10. Reconciliation and evaluation are unchanged.

## Sport sufficiency profile vocabulary

`SportSufficiencyProfile` defines:

- `Competition`
- `EnforcementMode`
- `RequiredGroups`
- `RecommendedGroups`
- `OptionalGroups`
- `IgnoredGroups`
- `MinimumBand`
- `AllowPrimaryFulfillment`
- `AllowProbeFallback`
- `Notes`

Enforcement vocabulary:

- `off`
- `observed`
- `advisory`
- `soft_enforced`
- `hard_enforced`

Only `observed` is used live for MLB in this slice. Unsupported competitions return `off`. Advisory, soft-enforced, and hard-enforced are vocabulary only.

## Current MLB observed profile

- `Competition`: `mlb`
- `EnforcementMode`: `observed`
- `RequiredGroups`: `starting_pitching`
- `RecommendedGroups`: `market_odds`, `bullpen_availability`, `lineup_injury`, `weather_park`, `market_movement`
- `OptionalGroups`: `identity_schedule`, `team_form`, `rest_travel`
- `IgnoredGroups`: `fallback_proxy`
- `MinimumBand`: `thin`
- `AllowPrimaryFulfillment`: `true`
- `AllowProbeFallback`: `false`

MLB does not require `moderate` or `rich` coverage. `market_odds` remains recommended and future-candidate only for MLB; it does not satisfy an active required path.

## How the catalog informs the policy

`PerceiveFulfillmentProjection.Build(...)` reads the sport profile and checks `ProbeFallbackCatalog.Find(group, sport)` for each required group.

- Supported groups with a current primary tool id become `PrimaryFulfillmentGroups`.
- Probe fallback groups are only considered when the profile allows probe fallback.
- The catalog is metadata only; it does not authorize, schedule, or execute tools.
- `ProbeFallbackCatalogStatus.FutureCandidate` does not count as active support.

For MLB v1, `starting_pitching` can become a primary fulfillment group because the catalog has the `pitching.mlb.probable_starters` path. Probe fallback remains disabled.

## What is now live

`GET /api/agent-runs/{id}/artifact` now returns `PerceiveFulfillment` on the internal artifact DTO.

The value is derived at read time from:

- run competition.
- existing `SourceSufficiency`.
- sport sufficiency profile.
- `ProbeFallbackCatalog` metadata.

Expected MLB projections:

- thin sufficiency with grounded `starting_pitching` -> `FulfilledWithThinCoverage`, mode `observed`.
- missing `starting_pitching` with a supported primary path -> `PrimaryFulfillmentRequired`, mode `observed`.
- missing required group with no supported path -> `BlockedNotEvaluable` or equivalent policy result.
- `NoDirectionalSeparation` remains distinct from source insufficiency.

## What remains non-live

- no enforcement.
- no Probe execution.
- no Tool Gateway call.
- no model call.
- no prompt change.
- no analyzer behavior change.
- no confidence/posture/lean mutation.
- no calibration eligibility change.
- no reconciliation/evaluation change.
- no buyer display.
- no persistence or migration.

## Activation ladder

1. **Dormant:** policy and catalog exist, no runtime caller.
2. **Observed:** read/projection computes and exposes diagnostic metadata. This slice.
3. **Advisory:** internal workflows may warn, but still do not block.
4. **Soft enforced:** selected runs can be held or flagged under explicit policy.
5. **Hard enforced:** runs are blocked when fulfillment fails.

Do not skip observed telemetry. Enforcement requires evidence that the diagnostic result is stable and useful.

## Deferred

- advisory mode.
- soft enforcement.
- hard enforcement.
- persisted telemetry.
- tenant/product overrides.
- live Probe execution.
- Question structured trace.
- buyer display.
- WNBA support.
- source freshness rules.
- expanded primary source coverage.
- calibration impact review.

## Code and tests

Code added or updated:

- `platform/dotnet/DevCore.Api/AgentRuns/SportSufficiencyProfile.cs`
- `platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs`
- `platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs`
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/SportSufficiencyProfileTests.cs`
- `platform/dotnet/DevCore.Api.Tests/Integration/AgentRunsControllerTests.cs`

Tests cover:

- MLB profile is observed, not enforced.
- MLB `starting_pitching` is required.
- MLB `market_odds` remains recommended/future-candidate and not an active required path.
- artifact DTO projects observed fulfillment metadata.
- thin MLB coverage with grounded starting pitching projects `FulfilledWithThinCoverage`.
- missing MLB starting pitching with a primary source path projects `PrimaryFulfillmentRequired`.
- missing required group with no supported path projects `BlockedNotEvaluable`.
- projection does not mutate lean, confidence, posture, or `OutputJson`.
- normal run result DTO remains unchanged.

## Verification

- focused observed projection/profile tests: 9/9 passed.
- focused regression set (`SportSufficiencyProfileTests | PerceiveFulfillmentPolicyTests | ProbeFallbackCatalogTests | SourceSignalTaxonomyTests | ProbeRequestTests | artifact_endpoint`): 60/60 passed.
- full `DevCore.Api.Tests`: 705/705 passed.

No model calls, no sports generation, no reconciliation, no migrations, no Probe/tool execution, and no buyer/frontend changes.

## Recommended next slice

**Perceive Fulfillment Observability Review v1:** inspect a real batch of existing MLB artifacts through the new observed DTO field, compare decisions against artifact provenance and eventual reconciliation results, and decide whether advisory mode needs additional telemetry before any enforcement work.
