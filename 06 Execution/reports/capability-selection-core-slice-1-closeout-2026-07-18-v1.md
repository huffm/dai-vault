---
title: "Capability Selection Offline Core Slice 1 Closeout v1 (2026-07-18)"
type: "evidence-report"
date: "2026-07-18"
status: "complete"
project: "DAI"
slice: "WI-0031 Model-Assisted Capability Recommendation and Tool Selection v1 (Slice 1)"
repos:
  dai: "code+tests (offline domain core; additive; branch wi/0031-capability-selection-core)"
  dai-vault: "docs-only"
tags:
  - platform-architecture
  - tool-gateway
  - capability-selection
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
  - "06 Execution/plans/capability-recommendation-and-tool-selection-implementation-plan-v1.md"
---

# capability selection offline core slice 1 closeout v1

## purpose

Record WI-0031 Slice 1: the offline deterministic domain contracts and selection trace for the
Model-Assisted Capability Recommendation and Tool Selection standard, implemented and tested with no
i/o, no model call, no network, and no gateway dependency. This is the first WI-0031 implementation
slice; it delivers the pure functional core only.

## context

Executed on coordinated branches from the integrated heads: dai `wi/0031-capability-selection-core`
from `c6166e2`, dai-vault `wi/0031-capability-selection-core` from `d6eef7d`. Gated behind the
WI-0032 knowledge-architecture standard disposition (Slices 1-2 integrated) per the recorded
sequence.

## language / policy-authority decision (the gating step)

**C#/.NET.** The deterministic authority WI-0031 reuses is the .NET Tool Gateway
(`platform/dotnet/DevCore.Api/Tools/ToolDefinition.cs`: `AllowedProtocolNodes`, `ToolKind`,
`ToolTransport`, `ToolIdempotency`, `ToolCostClass`; `ToolGateway`/`IProtocolToolAccessPolicy`).
There is no Python capability-selection authority. The pure domain core therefore lives in the
existing `DevCore.Domain` project (`net10.0`, nullable enabled, no i/o), which `DevCore.Api`
references and `DevCore.Api.Tests` (xUnit) sees transitively -- a bounded path needing no new
project or csproj change. Idiom mirrors the existing pure deriver pattern
(`AdvertisedStrengthDeriver`).

## scope

Included: the offline domain contracts (disposition enum, ingredient/gate/score/candidate/trace
records), the pure `BuildTrace` core, canonical JSON serialization, and 7 xUnit tests; the WI-0031
spec Slice-1 disposition; this closeout; the current-slice append. Excluded: the model call, the
capability/tool registry and resolution (Slice 2), the recommender (Slice 3), real hard gates +
ranking + recipe compilation (Slice 4), telemetry persistence (Slice 5), any pilot integration, and
any CLI/service/network.

## key decisions and invariants proven

1. **Model recommends, deterministic policy selects.** The core takes resolved candidates as inputs
   and shapes them into a versioned trace; it authorizes nothing (`ScreeningAuthorized` and
   `CaptureAuthorized` are always `false`).
2. **Eligibility precedes ranking; no score rescues a failed hard gate.** A candidate with any failed
   required `HardGateResult` is ineligible with disposition `ModelRecommendationRejectedByPolicy` and
   `FinalScore == 0`, even with a higher semantic relevance -- proven by
   `failed_hard_gate_is_never_selected_even_with_higher_score_and_is_retained_as_shadow`.
3. **Relevant-but-inaccessible is retained as shadow, never selected** -- proven by
   `inaccessible_candidate_is_retained_as_shadow_never_selected`.
4. **Deterministic output.** Inputs are ordinally ordered and the best eligible candidate per
   capability is selected with a stable tool-id tie-break; identical inputs yield byte-identical
   canonical JSON regardless of input order -- proven by the serialization and order tests.

## evidence

- review correction (2026-07-18, dai commit `69cee8b`, WI: WI-0031): selection is now keyed by
  `(capability, tool)` rather than by tool id alone, so a tool that is the best choice for one
  capability is no longer accidentally marked `Selected` under another capability where it is merely
  eligible; added a regression test plus an empty-candidate-set test. This corrected a determinism/
  correctness defect found in review before integration.
- files (dai, additive; branch `wi/0031-capability-selection-core`, tip `69cee8b` = `5def141` core +
  `69cee8b` review correction):
  `platform/dotnet/DevCore.Domain/CapabilitySelection/CapabilitySelectionCore.cs` (new),
  `platform/dotnet/DevCore.Api.Tests/CapabilitySelection/CapabilitySelectionCoreTests.cs` (new). No
  existing source modified; no csproj change; the documented DevCore.Data.csproj phantom untouched.
- build: `DevCore.Domain` builds 0 warnings / 0 errors.
- tests: targeted 7/7 pass; full `DevCore.Api.Tests` suite **1286 passed / 0 failed / 0 skipped**
  (1279 prior + 7 new). Pre-existing warnings (NU1903 OpenApi advisory; CS warnings in
  TestAuthentication/BuyerArtifactProjection/ProtocolFailureSeam) are not from the new files.
- pre-write gate: dai `c6166e2` 0/0, vault `d6eef7d` 0/0; drift classified + disjoint; strict
  snapshot exit 0 / 0 warnings; branches created before first write from the verified heads.

## safety / non-actions

0 model/paid/source-readiness calls; 0 services started; 0 database reads/writes; 0 application-data
writes; 0 network; 0 runtime wiring (the core is not called by any endpoint yet); 0 prompt/routing/
confidence/calibration/buyer/schema/migration changes; 0 gateway or permission changes; 0
pushes/merges. The core has no i/o and no gateway dependency; it authorizes nothing.

## files created/changed (this slice)

- dai (branch `wi/0031-capability-selection-core`, tip `69cee8b`): the two new .cs files above
  (`5def141` core + `69cee8b` review correction).
- vault (branch `wi/0031-capability-selection-core`): WI-0031 spec Slice-1 disposition; this
  closeout; append-only `06 Execution/handoffs/current-slice.md`.

## next step

WI-0031 Slice 2 (tenant-scoped capability and tool registry resolution) under separate authorization:
resolve capabilities to tenant-approved `ToolDefinition` implementations, expose accessible and
shadow catalogs, enforce role/station/tenant scopes, retain inaccessible recommendations -- reusing
the Tool Gateway. Review + integration of this Slice 1 (dai + vault branches) is a separate gated
step. A recommendation is not an authorization.
