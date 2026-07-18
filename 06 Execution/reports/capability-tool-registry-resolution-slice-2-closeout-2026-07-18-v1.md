---
title: "Capability/Tool Registry Resolution Slice 2 Closeout v1 (2026-07-18)"
type: "evidence-report"
date: "2026-07-18"
status: "complete"
project: "DAI"
slice: "WI-0031 Model-Assisted Capability Recommendation and Tool Selection v1 (Slice 2)"
repos:
  dai: "code+tests (deterministic resolver + gateway adapter; additive; branch wi/0031-capability-tool-registry-resolution)"
  dai-vault: "docs-only"
tags:
  - platform-architecture
  - tool-gateway
  - capability-selection
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
  - "02 Platform/architecture/tool-gateway-and-agent-permissions-doctrine-v1.md"
---

# capability/tool registry resolution slice 2 closeout v1

## purpose

Record WI-0031 Slice 2: deterministic tenant-scoped resolution of capability recommendations to
known tool implementations, offline, consuming the canonical Tool Gateway policy/registry seams
without creating a second permission authority and without invoking any tool.

## context

Coordinated branches from the integrated Slice-1 heads: dai
`wi/0031-capability-tool-registry-resolution` from `69cee8b`, vault same-named branch from
`c6e881d`. No model call, persistence, endpoint, or live execution.

## policy-authority decision

The canonical enforced access seam is `IProtocolToolAccessPolicy.IsAllowed(ToolDefinition,
protocolNode)` (`platform/dotnet/DevCore.Api/Protocols/ProtocolToolAccessPolicy.cs`) -- non-trivial:
it consults station cards (`AllowedTools`) plus stage sentinels plus `AllowedProtocolNodes`.
Re-implementing it would create a second, divergent permission authority, so the resolver
**consumes** it (and `IToolRegistry.TryGet`) via a thin adapter; it never recomputes it.
`DevCore.Domain` is pure (no `DevCore.Api` reference), so the resolver takes access-policy results
as supplied facts; the `DevCore.Api` adapter projects the real seams into those facts.
Tenant/role/side-effect/cost/rate/modality/schema enforcement is declarative/deferred in gateway v1
(only `AllowedProtocolNodes` is enforced; tool-gateway doctrine), so those facts stay at their
allowed defaults in the adapter until a canonical seam exists -- the resolver never invents them.
`ARCHITECTURE_BLOCKED:*` not triggered: a canonical seam exists and no project-file change was
needed.

## scope

Included: the pure Domain resolver + contracts + Slice-1 projection; the thin gateway adapter; 21
tests; the WI-0031 Slice-2 disposition; this closeout; the current-slice append. Excluded: model
recommendation (Slice 3), hard gates + ranking + recipe (Slice 4), telemetry persistence (Slice 5),
pilot integration (Slice 6); registry persistence, endpoints, DI wiring, live tool execution,
production weighting, network.

## contracts introduced

`CapabilityRecommendationInput`, `CapabilityRegistration` (identity `(capability, tool)`),
`ToolDefinitionFacts` (domain projection of ToolDefinition existence/config/operational facts),
`CandidateAccessFacts` (supplied canonical access results), `ResolutionContext` (tenant/role/node +
versions), `ResolvedCandidate` (Disposition null when accessible; a normative shadow disposition +
ordinal reason otherwise), `ResolutionResult` (accessible + shadow catalogs + deterministic
`RejectedInputs`). `CapabilityToolResolver.Resolve` is pure and deterministic;
`ToToolCandidateInputs` projects into the Slice-1 core.

## key decisions and invariants proven

1. **Consume, do not duplicate.** The adapter calls `IProtocolToolAccessPolicy.IsAllowed` and
   `IToolRegistry.TryGet`; the resolver organizes the results and authorizes nothing.
2. **No score rescues any denial; denied candidates are never accessible and never selectable.**
   Fixed-precedence access evaluation (tenant -> role -> protocol node -> side-effect -> cost ->
   rate -> modality -> schema); denials go to the shadow catalog; the Slice-1 projection marks them
   `Accessible=false` so `BuildTrace` retains them as shadow and cannot select them (composition
   test).
3. **Accessible and shadow catalogs are mutually exclusive.**
4. **Capability identity distinct from tool identity;** the same tool may implement several
   capabilities and a capability may map to several tools, all keyed `(capability, tool)`.
5. **Deterministic.** Ordinal ordering of inputs and outputs; identical output regardless of input,
   dictionary, or hash order; no culture/time/random/process/env dependence.
6. **Malformed/duplicate handling is explicit, never silent:** blank capability/tool ids rejected;
   non-finite scores rejected; duplicate `(capability, tool)` registrations and duplicate capability
   recommendations deduped and recorded in `RejectedInputs`; unmapped capabilities and missing tool
   definitions retained as shadow.
7. **Tenant isolation:** the same registry under two contexts yields different accessible/shadow
   results from the supplied per-context facts.

## evidence

- files (dai, additive; branch `wi/0031-capability-tool-registry-resolution`, commit `b31cc69`):
  `platform/dotnet/DevCore.Domain/CapabilitySelection/CapabilityToolResolver.cs` (new),
  `platform/dotnet/DevCore.Api/CapabilitySelection/CapabilityResolutionGatewayAdapter.cs` (new),
  `platform/dotnet/DevCore.Api.Tests/CapabilitySelection/CapabilityToolResolverTests.cs` (new),
  `platform/dotnet/DevCore.Api.Tests/CapabilitySelection/CapabilityResolutionGatewayAdapterTests.cs`
  (new). No existing source modified; no `.csproj`/`.slnx` change; csproj phantom untouched.
- build/tests: `DevCore.Domain` builds 0 warnings; full `DevCore.Api.Tests` **1307 passed / 0
  failed / 0 skipped** (1286 prior + 21 new).
- pre-write gate: dai `69cee8b` 0/0, vault `c6e881d` 0/0; drift classified + disjoint; strict
  snapshot exit 0 / 0 warnings; branches created before first write from the verified heads.

## security findings

Cross-tenant registrations are not returned as accessible (access is per supplied context fact); a
caller cannot self-grant a role/station permission (access is a supplied fact, not an input field
the resolver trusts as authority); provided facts are keyed to the resolved `(capability, tool)`;
capability mappings expose no credentials; `ToolDefinition` secrets/transport config are not
serialized (the domain projection carries only id/existence/config/operational booleans); shadow
output contains only ids + reason codes; no secret/token/connection-string/tenant payload enters
resolution output.

## safety / non-actions

0 model/paid/source-readiness calls; 0 services; 0 database reads/writes; 0 application-data writes;
0 network; 0 live tool calls; 0 gateway execution; 0 permission-authority changes; 0 endpoint/DI
wiring (the resolver and adapter are not registered in DI or called by any route); 0 persistence; 0
schema/migration; 0 project-file change; 0 pushes/merges.

## next step

WI-0031 Slice 3 (model-assisted ingredient and capability recommender) under separate authorization:
a strict model input/output schema, versioned prompt, metering, capability recommendation +
uncertainty + unmapped-capability output -- the model still has no execution authority and produces
`CapabilityRecommendationInput` values that feed this resolver. Review + integration of this Slice 2
is a separate gated step. A recommendation is not an authorization.
