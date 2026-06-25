# cognitive factory read-only execution v1

**status:** active doctrine -- implemented (Stage 2 of the activation ladder). dev-only, config-gated, audit-only,
non-mutating. the first controlled cognitive-factory runtime execution.
**date:** 2026-06-25
**type:** platform runtime architecture slice (Factory Maturity Phase II, Activation Ladder Stage 2). TDD;
dai-code-reviewer (clean); full suite 928/0.

## purpose

Prove the Cognitive Factory can safely enter and exit a controlled runtime execution path -- not improve a decision.
Stage 0 gave observability, Stage 1 gave config-bound control; Stage 2 runs exactly one deterministic, audit-only,
non-mutating probe behind config gates, so execution produces observability, not product behavior.

> Stage 2 permits execution only when execution produces observability, not product behavior.

## endpoint

`POST /api/dev/cognitive-factory/probe/audit-only`

- **development-only.** 404 in any non-Development environment (gated by `!env.IsDevelopment()`, checked before any
  config read or execution), even with valid config.
- **anonymous within development is intentional** (mirrors the diagnostics endpoint): the path executes only the
  deterministic, non-mutating probe and exposes no tenant data or secrets; the dev-only gate is the control.

## config gates (strict)

Execution proceeds only when ALL hold (pure `CognitiveFactoryExecutionGate.Evaluate`):

- `CognitiveFactory:Enabled = true`
- `CognitiveFactory:AllowExecution = true`
- `CognitiveFactory:AuditOnly = true`
- `CognitiveFactory:AllowArtifactMutation = false`

Any missing/unsafe gate refuses with a `200` structured result naming the blocking gate (refusal is a designed
outcome, not an error). Missing config resolves to default-off -> refused. Gates are never inferred or relaxed.

## execution scope

When permitted, the endpoint runs **exactly one** deterministic protocol-node probe:
`ProtocolNodeRunner.ExecuteAsync(interrogate.probe)`. That path (verified in source + by code review):

- builds a deterministic probe via `CognitiveProtocolBuilder.BuildProbe` / `BuildProbeRequest`;
- makes **no model call, no tool/gateway call, no external api call**;
- **persists nothing**; artifact mutation is structurally impossible on this path;
- a non-`interrogate.probe` station returns `UnsupportedStation`, an unknown id `UnknownStation` -- neither executes
  anything. The controller hardcodes `interrogate.probe`.

It answers "can the Cognitive Factory safely enter and exit a controlled audit-only runtime path?" -- not "can the
system improve the decision?"

## what it returns (CognitiveFactoryExecutionResult)

`executed`, `refused`, `refusalReason`, `executionMode` ("audit-only"), `stationId`, `executionStatus`,
`probeOutputPresent`, `auditRecordId` (null), `auditPersisted` (false), `artifactMutated`/`decisionMutated`/
`modelCalled`/`externalToolCalled`/`dbWritten` (all false -- truthful, the path cannot do them), `elapsedMs`,
`warnings`, `blockingRequirementBeforeStage3`, `notes`.

## audit persistence behavior (Part G)

**No DB write.** The existing audit infrastructure (`ProbeRefreshMergeAuditStore` -> `ProbeRefreshMergeAudits`) is
**merge-specific** -- its schema (before/after, protected fields, idempotency key) cannot honestly represent a
protocol-node probe. Per the slice's rule ("if existing schema cannot safely represent this, do not write to DB;
return an in-memory audit result and document the limitation"), the result is **in-memory only**
(`auditPersisted = false`, `dbWritten = false`). Durable, non-decision audit persistence is deferred to a future
slice that adds a fit-for-purpose ledger.

## what it can / cannot write

- **can write:** nothing durable in v1 (in-memory result only).
- **cannot write:** artifacts, AgentRuns, outcomes, evaluations, calibration rows, the merge-audit ledger, or any
  decision data. Verified by tests asserting `AgentRunOutcomes`/`AgentRunEvaluations`/`ProbeRefreshMergeAudits`
  counts stay 0 after a permitted execution.

## diagnostics integration

`GET /api/dev/cognitive-factory/diagnostics` now reports `activationStage` 2 ("Read-Only Execution") and a
`ReadOnlyExecution` block: `endpointExists`, `endpointPath`, `auditOnlyExecutionPermittedByConfig` (the pure gate
read), `executionMode`, and `artifactMutationBlocked` / `toolExecutionBlocked` / `externalCallsBlocked` /
`modelCallsBlocked` (all true), `dbWritesByExecution` (false), `blockingRequirementBeforeStage3`. `cognitiveRuntime
Activated` stays false. Computing this block executes nothing -- the gate is a pure config read.

## relationship to Stage 0 and Stage 1

- Stage 0 (`cognitive-factory-observability-surface-v1.md`): the read-only diagnostics surface this slice extends.
- Stage 1 (`cognitive-factory-configuration-bound-control-v1.md`): bound the config posture; Stage 2 adds the first
  consumer (`AllowExecution` + `AuditOnly`) that reads it -- but only to run the inert probe. Stage 1's "what
  remains before Stage 2" is now done (reconciled there).

## what remains before Stage 3 (controlled mutation)

A merge writer + a rollback executor (neither exists), plus calibration sufficiency -- Stage 3 is **locked behind
Evidence Readiness Gate 4/5** (Gate 4 not achieved; see `02 Platform/architecture/governance/evidence-readiness-
gates-v1.md`). Also: a durable, non-decision audit ledger for the audit-only path (deferred this slice).

## verification evidence

- TDD: gate red->green (4 refusal tests failed against an always-permit stub); endpoint red->green (execution tests
  failed against a NotFound stub controller + wrong diagnostics block).
- Tests (13 new): gate refusal/permit (5); prod 404 even with valid config; refused when default-off; refused when
  AllowArtifactMutation true; permitted audit-only writes nothing (outcomes/evaluations/merge-audits all 0);
  throwing `IToolGateway` still executes + returns 200 (no tool/external call); diagnostics reports Stage-2
  capability + permitted-when-safe + not-permitted-when-unsafe.
- Final verification: full `DevCore.Api.Tests` **928 / 0** (+13). `git diff --check` clean; comments lowercase
  ascii. dai-code-reviewer: clean -- all invariants verified (dev-first gate; inert execution path; truthful
  false-flags; no db write; diagnostics executes nothing).

## related docs

- `cognitive-factory-runtime-activation-readiness-v1.md` -- the activation ladder.
- `cognitive-factory-observability-surface-v1.md` (Stage 0), `cognitive-factory-configuration-bound-control-v1.md`
  (Stage 1).
- `02 Platform/architecture/governance/evidence-readiness-gates-v1.md` -- Stage 3 (mutation) is a Gate 4/5 activity.

## recommended next slice

Not Stage 3 (locked by Gate 4/5). Either **Cognitive Factory Audit Ledger v1** (a fit-for-purpose, non-decision
durable audit record for the audit-only execution path, dev-only, excluded from calibration/settlement), or resume
the evidence track (Directional-Contrast Cohort Capture v3 + Market Baseline v2) to work toward Gate 4.
