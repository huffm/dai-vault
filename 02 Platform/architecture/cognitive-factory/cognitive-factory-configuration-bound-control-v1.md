# cognitive factory configuration-bound control v1

**status:** active doctrine -- implemented (Stage 1 of the activation ladder). config-bound, default-off, observable,
not executable.
**date:** 2026-06-25
**type:** platform runtime architecture slice (Factory Maturity Phase II, Activation Ladder Stage 1). code change is
limited to binding a default-off activation posture from `IConfiguration` and surfacing it on the existing dev-only
diagnostics endpoint; no runtime consumer was added. TDD; dai-code-reviewer; full suite 915/0.

## purpose

Make dormant cognitive-factory activation reversible through configuration before it is executable. Stage 0
(Observability) gave a read-only surface; Stage 1 makes the activation posture a config-bound, default-off,
observable value -- so a future switch becomes a reviewable config decision rather than a source edit. It binds and
observes the posture; it does **not** consume it to run anything.

> Stage 1 complete means runtime is configurable, not running.

## config section added/bound

A new strongly-typed options class `CognitiveFactoryActivationOptions`, bound from `IConfiguration` section
**`CognitiveFactory`** via `builder.Services.Configure<...>(...)` in `Program.cs`. Fields (all narrow, safe
default-off):

| field | default | meaning |
|---|---|---|
| `Enabled` | `false` | master activation posture |
| `AllowExecution` | `false` | would permit a dev-only run (no consumer reads it in Stage 1) |
| `AllowToolExecution` | `false` | would permit Tool Gateway handler invocation by an activated chain |
| `AllowArtifactMutation` | `false` | would permit artifact mutation (gated by Evidence Readiness Gate 4/5) |
| `AuditOnly` | `true` | when execution is eventually allowed, it is audit-only and non-mutating (most restrictive) |
| `ActivationStage` | `0` | operator-declared activation stage |
| `EnvironmentScope` | `"Development"` | environment the posture is intended to apply to |

There is no `EnableEverything`-style flag. The probe-refresh internal options (`ProbeRefreshExecutorOptions`,
`ProbeRefreshChainAssemblyOptions`, `ProbeRefreshArtifactMergeOptions`) remain compiled defaults; they are bound
only when a Stage-2 consumer exists to read them.

## default-off posture

- A **missing** `CognitiveFactory` section resolves to the C# defaults -- all `Allow*`/`Enabled` false. `Configure`
  registers `IOptions<>` unconditionally, so the posture is always bound and observable, just off.
- **explicit false** resolves off; **explicit true** is observable but inert (no consumer).
- Default-off holds in **every** environment (production, staging, development). No environment enables execution by
  default. Invalid/unknown config cannot activate runtime because no runtime path reads the config at all.

## diagnostics integration

`GET /api/dev/cognitive-factory/diagnostics` (dev-only, read-only) now reports a `ConfiguredActivation` block:

- `optionsBound`, `configSection` (`CognitiveFactory`), `configuredActivationStage`.
- the bound posture: `enabled`, `allowExecution`, `allowToolExecution`, `allowArtifactMutation`, `auditOnly`,
  `environmentScope` (observable).
- `executionBlocked`, `toolExecutionBlocked`, `artifactMutationBlocked` -- **hardcoded `true` in Stage 1**: no
  runtime path consumes the config, so execution stays blocked regardless of the configured values.
- `blockingRequirementBeforeStage2` and a note: "stage 1 complete means runtime is configurable, not running."

The top-level surface now reports `activationStage = 1` ("Configuration-Bound Control") while
`cognitiveRuntimeActivated` stays `false`. The "Feature Flags / Runtime Configuration" component flips to
`registered = true` (config-bound) with `enabled` reflecting the configured value and `readiness = PartiallyReady`
(config-bound but inert). The endpoint continues to execute nothing.

## what this does NOT activate

Executes no `ProtocolNodeRunner` / `DiscernStationRunner` / Probe Refresh chain; invokes no Tool Gateway handler,
external tool, or model; writes no database row; mutates no artifact / AgentRun / outcome / evaluation / calibration
row; enables no flag by default; adds **no consumer** that reads the config to run anything. Setting any config
value true changes only what the diagnostics surface reports -- it cannot make the runtime execute.

## relationship to Stage 0

Stage 0 (`cognitive-factory-observability-surface-v1.md`) built the read-only surface; Stage 1 reuses it to expose a
config-bound posture. The activation ladder
(`cognitive-factory-runtime-activation-readiness-v1.md`, Part E) is: Observability -> **Configuration-bound
control** -> Read-only execution -> Controlled mutation (Gate 4/5) -> Production. Mutating rungs remain gated by
`evidence-readiness-gates-v1.md` (Gate 4 not achieved).

## what remains before Stage 2 (read-only execution)

**done 2026-06-25** -- Stage 2 is implemented (`cognitive-factory-read-only-execution-v1.md`). _Today_ a dev-only
`POST /api/dev/cognitive-factory/probe/audit-only` reads `AllowExecution`+`AuditOnly` and runs one deterministic,
non-mutating protocol-node probe (no model/tool/external call, no db write). It writes no audit-ledger row -- the
existing merge-audit schema is merge-specific, so the result is in-memory only (durable audit deferred). The note
below was the Stage-2 plan when this Stage-1 doc was written.

A dev-only consumer that reads `AllowExecution` (and `AuditOnly`) to gate a single audit-only, non-mutating probe
run -- no artifact mutation, observable on the surface and reversible by the config flag.

## verification evidence

- TDD: red observed (stub `ConfiguredActivation` + un-bumped stage) before green.
- Tests (12 total; +4 this slice): missing config -> default-off; explicit false -> off; explicit true ->
  observable but every `*Blocked` stays true and the executor stays non-executable + `cognitiveRuntimeActivated`
  false; config-bound status reported; posture not observable outside development even when set true; plus the
  Stage-0 invariants (dev-only 404 in prod/staging, no db write incl. audit table, no construction/execution of the
  runtime).
- Final verification: full `DevCore.Api.Tests` **915 / 0**. `git diff --check` clean; comments lowercase ascii.
  dai-code-reviewer: invariants confirmed (no execution consumer; `*Blocked` literals; binder correct); findings
  applied (prod+true-config test; lowercase comments).

## related docs

- `cognitive-factory-observability-surface-v1.md` -- Stage 0 (reconciled: its "what remains before Stage 1" is now
  this doc).
- `cognitive-factory-runtime-activation-readiness-v1.md` -- the activation ladder + per-component readiness.
- `02 Platform/architecture/governance/evidence-readiness-gates-v1.md` -- mutating rungs (Stage 3+) are Gate 4/5.

## recommended next slice

**Cognitive Factory Read-Only Execution v1** (Stage 2): add the first dev-only consumer that reads `AllowExecution`
+ `AuditOnly` to run a single audit-only, non-mutating probe (one audit-ledger row, no artifact mutation),
reversible by config and observable here. Do not cross into mutation (Stage 3 / Gate 4/5).
