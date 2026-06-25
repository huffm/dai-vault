# cognitive factory observability surface v1

**status:** active doctrine -- implemented (dev-only read-only endpoint). Layer-0 / Stage-0 of the activation ladder.
**date:** 2026-06-25
**type:** platform runtime architecture slice (Factory Maturity Phase II, Activation Ladder Stage 0). Code change is
limited to one dev-only diagnostics endpoint; no cognitive runtime activated. TDD; dai-code-reviewer; full suite
911/0.

## purpose

Make activation observable before making activation possible. The Runtime Activation Readiness assessment named the
**observability gap** as a top architectural risk (diagnostics computed but surfaced nowhere; you cannot measure an
activation you cannot see) and named a read-only observability surface as the correct first activation. This slice
builds exactly that surface and nothing more. It strengthens operational readiness; it does not cross into runtime
execution and does not unlock tuning.

## endpoint

`GET /api/dev/cognitive-factory/diagnostics`

- **development-only.** Returns `404` in any non-Development environment (gated by `!env.IsDevelopment()`, mirroring
  `DevProvisionController`). Tested for Production and Staging.
- **anonymous within development is intentional** -- the payload is non-sensitive architecture topology (no tenant
  data, no secrets) and the endpoint executes nothing, so the dev-only gate is the security control. This differs
  from `DevProvisionController` (which mutates and therefore adds `[Authorize]` + a shared secret); a read-only dev
  surface does not warrant that friction. (Recorded as a deliberate decision, not an oversight.)

## what it reports

A `CognitiveFactoryDiagnostics` payload: `surfaceVersion`, `environment`, `generatedAtUtc`, `activationStage` (0),
`activationStageName` ("Observability"), `cognitiveRuntimeActivated` (false), `nextRecommendedStage`,
`overallReadiness`, `blockingPrerequisites`, `derivedFrom`, and a list of components. Each component reports
`name`, `category`, `implemented`, `registered`, `enabled`, `executable`, `activationLayer`, `readiness`
(`Ready` / `PartiallyReady` / `NotReady` / `LivePartial`), `status`, `blockingRequirements`, `notes`.

13 components surfaced: Protocol Registry, Protocol Validator, Protocol Node Runner, Discern Station Runner, Tool
Gateway, Protocol Tool Access Policy, Probe Refresh Chain Assembly, Probe Refresh Executor, Probe Merge Contract,
Probe Audit Store / Read Surface, Runtime Diagnostics, Feature Flags / Runtime Configuration, Dependency Injection
Registration.

**Sources of truth (`derivedFrom`):**

- **di registration probe** -- `IServiceProviderIsService.IsService(type)`, which reports whether a component is
  registered **without constructing it**, so nothing is ever instantiated or executed.
- **compiled option defaults** -- the real `ProbeRefreshExecutorOptions.Disabled.Enabled`,
  `ProbeRefreshChainAssemblyOptions.Disabled.Enabled`, `ProbeRefreshArtifactMergeOptions.Disabled.ArtifactMergeEnabled`
  constants ground the `enabled` flags (not magic strings).
- **static inventory** -- the per-component facts that have no safe runtime self-inspection ("has a live caller",
  intended readiness) are encoded as a verified inventory traced to
  `cognitive-factory-runtime-activation-readiness-v1.md`, and labelled as such in `derivedFrom`.

## what it explicitly does NOT do

Executes no `ProtocolNodeRunner` / `DiscernStationRunner` / Probe Refresh chain; calls no Tool Gateway handler, no
external tool, no model; writes no database row (verified against `AgentRunOutcomes`, `AgentRunEvaluations`, and the
probe-refresh `ProbeRefreshMergeAudits` table); mutates no artifact / AgentRun / outcome / calibration row; enables
no feature flag; changes no settlement, reconciliation, buyer, or frontend behavior. It constructs no cognitive
component (registration is probed, not resolved).

## activation ladder stage + relationship to readiness v1

This is **Stage 0 (Observability)** of the activation ladder defined in
`cognitive-factory-runtime-activation-readiness-v1.md` (Part E, Layer 0). It is the prerequisite the readiness doc
and the deferred-runtime ledger named for any later activation: a surface to watch a component on before it is
switched on. It deliberately stops at observation -- it binds no config and executes nothing.

The readiness doc's "Runtime Diagnostics ... surfaced by no endpoint" and the "observability -- Weak (runtime)"
maturity line are now partially superseded: the diagnostics are surfaced (read-only, dev-only). That doc has been
reconciled with an Originally/Today note.

## verification evidence

- TDD: red observed (component assertions failed against an empty-inventory stub) before green.
- Tests (8): dev returns payload + writes no db rows (incl. the audit table); Production 404; Staging/non-dev 404;
  Tool Gateway reported `LivePartial` + registered + executable; Probe Refresh Executor `enabled=false` +
  `NotReady`; Discern Station Runner `registered=false`; Feature Flags `registered=false` + `enabled=false`; the
  diagnostics path does not execute the runtime (throwing node-runner and a throwing factory for the
  gateway-capable executor both leave the request at 200, proving no construction/execution).
- Final verification: full `DevCore.Api.Tests` **911 / 0** (+8). `git diff --check` clean; added comments lowercase
  ascii. dai-code-reviewer run; findings triaged (auth decision documented; convention + 3 test tightenings
  applied).

## what remains before Stage 1 (configuration-bound activation)

- Bind `ProbeRefreshChainAssemblyOptions` / `ProbeRefreshExecutorOptions` from `IConfiguration` (default off), so a
  switch becomes a reviewable, reversible config decision instead of a source edit, and surface the bound value on
  this endpoint. No execution -- defaults stay off.
- Only after Stage 1 does Stage 2 (dev-only, audit-only, non-mutating probe run) become appropriate; Stage 3
  (controlled mutation / merge writer) remains gated by Evidence Readiness Gate 4/5 (not met).

## related docs

- `02 Platform/architecture/cognitive-factory/cognitive-factory-runtime-activation-readiness-v1.md` -- the
  assessment that defines the activation ladder; this slice implements its Stage 0.
- `02 Platform/architecture/governance/evidence-readiness-gates-v1.md` -- mutating activation (Stage 3+) is a Gate
  4/5 activity; this read-only surface is not.

## recommended next slice

**Cognitive Factory Configuration-Bound Control v1** (Stage 1): bind the probe-refresh options from configuration
(default off) and surface the bound values here. No execution. Defer Stage 2 to a subsequent slice.
