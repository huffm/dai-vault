# cognitive factory runtime activation readiness v1

**status:** active doctrine -- read-only architecture readiness assessment; no runtime activated
**date:** 2026-06-25
**type:** architecture readiness slice. all claims verified against runtime source + DI + config (not docs alone);
file:line citations throughout. no code, flag, wiring, or behavior change.

## purpose

Decide whether the dormant Cognitive Factory runtime is architecturally ready for a first controlled activation,
and if so what to activate first. This assesses prerequisites, contracts, invariants, and rollback boundaries; it
does not activate anything. Activation of a mutating/decision path is itself gated by
`02 Platform/architecture/governance/evidence-readiness-gates-v1.md` -- a runtime that can change a decision is a
Gate 4/5 activity, and Gate 4 is not yet achieved.

## headline finding

**update 2026-06-25:** Stage 0 (Layer-0) observability is now implemented -- a dev-only read-only diagnostics
endpoint (`cognitive-factory-observability-surface-v1.md`). The statements below that the runtime diagnostics are
"surfaced by no endpoint" and that observability is "Weak (runtime)" predate that slice; they describe the state
this assessment found, which the observability surface has since begun to address (read-only; no runtime activated).

The premise "the whole factory is dormant" is half right. There are **two distinct bodies of runtime**:

1. **The Tool Gateway is LIVE** -- it is on the critical path of every sports run today, enforcing fail-closed
   authorization and per-invocation audit logging.
2. **The Protocol stations + the 17-seam Probe Refresh chain are dormant** -- DI-wired but called by nothing, with
   every master switch defaulting OFF and **none bound to configuration**, so activation today requires a source
   edit, not a config flip.

**Answer: NO, the dormant runtime is not ready for a *mutating* first activation** -- but a low-risk *read-only*
first activation (observability) is available and is the correct first step. The single capability that most blocks
safe activation is **runtime observability + a config-bound reversible flag**: today a dormant component cannot be
turned on by a reviewable, measurable, reversible switch, and its state is not surfaced anywhere an operator can see.

## verification basis (independently confirmed this slice)

- Tool Gateway wired at `Program.cs:164` (`AddDaiToolGateway()`); live callers `SportsRetriever.cs:67/77/89/104`,
  `SportsAnalyzer.cs:53`, `SportsReferenceController.cs:198`, on the orchestrated path `AgentRunService.cs:43/52`.
- `ProbeRefreshExecutor` registered `ProbeRefreshExecutorOptions.Disabled` (`ToolGatewayServiceCollectionExtensions.cs:61-63`);
  `ProbeRefreshChainAssemblyOptions` defaults `Enabled=false, AllowGatewayExecution=false, PersistAuditRecord=false`
  (`ProbeRefreshChainAssembly.cs:76-78`).
- `DiscernStationRunner` has **no DI registration** (grep empty); no `ProbeRefresh` config section or binding (grep
  empty); no protocol/probe/station HTTP routes (grep empty).

## A. runtime inventory

Status legend: implemented (code exists) / wired (in DI) / executable (a live path can reach it) / dormant
(no live caller or default-off) / risk (if activated).

### A.1 Tool execution (LIVE)

| component | purpose | implemented | wired | executable (live) | dormant | depends on | prod risk | readiness |
|---|---|---|---|---|---|---|---|---|
| `ToolGateway` / `IToolGateway` | single authority for every tool call; authz + audit, then dispatch to a keyed handler | yes | yes (scoped, `Extensions.cs:56`) | **yes -- every sports run** | no | ToolRegistry, ProtocolToolAccessPolicy, handlers | already live; behavior unchanged from direct client + authz/telemetry | PARTIALLY READY (live; 2 of 5 guarantees enforced) |
| `ProtocolToolAccessPolicy` | authorize a tool for a node; fail-closed | yes | yes (singleton) | yes (sentinel branch only) | station-id branch dormant | ProtocolRegistry | low | READY (sentinel path) |
| tool handlers (10) | thin wrappers over typed clients | yes | yes (keyed scoped) | yes (per niche) | no | typed HTTP clients | low | READY |

Tool Gateway enforced **now**: authorization (`ToolGateway.cs:59-72`, fail-closed), per-call audit log event 1001.
**Not** enforced (declared but deferred): rate-limit / cost-class, idempotency caching, gateway-level timeout,
tenant-tier, durable audit persistence on the live path, outbound correlation-header injection, the station-id
authorization branch (`ProtocolToolAccessPolicy.cs:50-59`).

### A.2 Protocol runtime core (groundwork; mostly dormant)

| component | purpose | implemented | wired | executable (live) | dormant | risk | readiness |
|---|---|---|---|---|---|---|---|
| `ProtocolRegistry` (+Validator, +Extensions) | static manifest of 15 station cards; startup fail-fast validation | yes | startup guard `Program.cs:169` | validation runs every boot | data unused on run path | very low (boot check) | READY (as data + guard) |
| `ProtocolNodeRunner` | resolve station by id; `ExecuteAsync` supports ONLY `interrogate.probe` | yes | yes (singleton) | no (diagnostics/tests only) | yes | low (no model/tool/gateway call) | PARTIALLY READY (probe path safe; no caller) |
| `DiscernStationRunner` | dormant deterministic Discern-input shape | yes | **NOT registered** | no (tests only) | fully | low intrinsically; needs DI+caller | NOT READY |
| `ProtocolStationCard` / `ResultEnvelope` / `StepTrace` | machine-readable contracts/value shapes | yes | as data | no | yes | none (pure) | READY (as contracts) |
| `ProtocolStationDiagnostics` / `ProtocolDiagnosticsRollup` | read-only health inspection of station/chain state | yes | yes (singleton) | computed, **surfaced by no endpoint** | yes (no consumer) | none (read-only) | READY (logic); NOT SURFACED |

Dormancy anchor: every live caller passes a **stage sentinel** (`platform.retrieve/analyze/reference`), never a
canonical station id, so the registry-backed station path never executes. `RuntimeMaturity.Mature` on a station card
is **diagnostics metadata, not an activation switch** (`ProtocolStationCard.cs:75`) -- do not read "Mature" as
"ready to wire".

### A.3 Probe Refresh chain (largest dormant subsystem; 17 seams)

DI-wired end to end (`Extensions.cs:47-114`) but **no live caller** -- only tests and the read-only rollup inspect
results. Five independent default-OFF master switches; four inert mutation-intent flags with no reader/writer.

| switch | default | gates |
|---|---|---|
| `ChainAssemblyOptions.Enabled` | false | the whole chain (returns Disabled before any seam) |
| `ChainAssemblyOptions.AllowGatewayExecution` | false | the only path to a live fetch |
| `ProbeRefreshExecutorOptions.Enabled` | false (registered Disabled) | second independent gate on the executor |
| `ChainAssemblyOptions.PersistAuditRecord` | false | the only DB write (audit ledger row) |
| `ArtifactMergeOptions.ArtifactMergeEnabled` | false | merge **planning** only |
| `AllowArtifact/Confidence/Posture/LeanMutation` | false (inert) | document the protected-field boundary; no reader |

- **Tool Gateway call: impossible in production** -- needs both `AllowGatewayExecution=true` and executor
  `Enabled=true`; executor is injected Disabled (`Extensions.cs:63`); guard `ProbeRefreshExecutor.cs:116`.
- **Artifact mutation: structurally impossible** -- no writer exists anywhere; the merge step yields only a plan
  (`ProbeRefreshArtifactMergeContract.cs:125`), dry-run yields a projection (`IsFullArtifactCopy:false`).
- **Only DB write: audit ledger** (`ProbeRefreshMergeAuditStore.cs:84-108`), gated by `PersistAuditRecord`,
  idempotent; never artifacts. Protected fields (confidence/posture/lean) blocked at four layers (planner, review,
  dry-run filter, diagnostics).

Readiness: **NOT READY** for any production activation -- no live caller, no config-bound flag, no merge
writer/rollback executor, no runtime telemetry/operator surface, no tenant/economic gate, no calibration proof.

### A.4 Wiring / flags / config (the reachability constraint)

- All registrations live in `AddDaiToolGateway` (`Program.cs:164`) + the startup validator (`:169`).
- **No probe-refresh/chain/protocol option is bound from `IConfiguration`.** Every switch is a compiled
  record/static default. `"ProbeRefresh:ArtifactMergeEnabled"` is a **provenance label string**, not a live config
  key; appsettings has no `ProbeRefresh` section. -> Activation is **not reachable via appsettings**; it is a source
  edit today.
- **No protocol/probe/station/chain HTTP endpoint exists.** The audit read service is registered but injected into
  no controller. The entire dormant surface is unreachable over HTTP.

## B. activation dependency graph

```
[ Tool Gateway: LIVE ] (authz + audit enforced)
        |  (already activated for the current pipeline)
        v
Layer 0  Observability surface (read-only)
  ProtocolStationDiagnostics + ProbeRefreshChainDiagnostics + ProtocolDiagnosticsRollup
  depends on: a (dev-gated) read endpoint to surface already-computed state. NO execution.
        |
        v
Layer 1  Config-bound reversible flags
  bind ChainAssemblyOptions / ProbeRefreshExecutorOptions from IConfiguration (default false)
  depends on: Layer 0 (so the flag's effect is visible). Still NO execution (defaults false).
        |
        v
Layer 2  Probe execution -- dev only, audit-only, NON-mutating
  Enabled + AllowGatewayExecution + Executor.Enabled + PersistAuditRecord, ONE niche/tool, DEV
  depends on: Tool Gateway (live), Layer 1 flag, the audit store (exists), Layer 0 to observe it
        |
        v
Layer 3  (FUTURE, gated) merge writer + rollback executor -> artifact mutation
  depends on: Layer 2 evidence + Evidence Readiness Gate 4 (calibration sufficiency) and Gate 5,
  neither achieved. NOT recommended now.
```

Component-level prerequisites (verified):

- **ProtocolNodeRunner (probe)** needs: a live read-only caller + Layer-0 observability. No tool/model call, so low
  risk; but no caller exists.
- **Probe Refresh executor** needs: config-bound flag (Layer 1) + observability (Layer 0) + Tool Gateway (have) +
  the audit store (have). Its activation is the first that produces a real external fetch.
- **Merge writer** needs: a writer + rollback executor (neither exists) + calibration proof for any
  confidence/posture/lean touch (forbidden today) -> Gate 4/5.

## C. readiness classification (grounded)

| component | classification | why (grounded) |
|---|---|---|
| Tool Gateway (current pipeline) | **READY / already live** | authz + audit enforced; fail-closed; startup drift guard |
| Tool Gateway (full cloud-runtime vision) | **PARTIALLY READY** | rate-limit, idempotency, timeout, tenant-tier, durable audit, correlation header all absent/deferred |
| ProtocolRegistry + validator | **READY** | data + fail-fast boot check; no run-path dependency |
| Protocol station contracts/value shapes | **READY** | pure records; unused live |
| ProtocolNodeRunner (probe path) | **PARTIALLY READY** | executable + safe, but no live caller |
| Protocol station runtime (station-id path) | **NOT READY** | no caller passes a station id; sentinel-only today |
| DiscernStationRunner | **NOT READY** | not in DI; tests only; doctrine anti-goal |
| Probe Refresh chain (audit-only, dev) | **PARTIALLY READY** | reachable by constructing enabled options in dev; safe-by-default; but no config flag, no caller, no telemetry surface |
| Probe Refresh chain (production) | **NOT READY** | no live caller, no config binding, no telemetry/operator surface, no tenant gate |
| Merge writer / artifact mutation | **NOT READY** | no writer exists; structurally impossible; Gate 4/5 unmet |
| Runtime observability surface | **NOT READY** | diagnostics computed but surfaced by no endpoint; telemetry is log-only; no metrics |

## D. activation invariants (template for every future activation)

Every future activation MUST define and satisfy:

- **Preconditions:** the lower dependency layer is active and observed; the switch is config-bound and defaults OFF;
  the activation is scoped (dev first, one niche/tool, one component).
- **Success criteria:** a measurable, pre-stated signal (e.g. "one audit ledger row written with the expected
  before/after and no protected-field change"; "diagnostics report `EnabledButNonMutating`").
- **Observability:** the component's state and outcome are visible BEFORE flipping the switch (Layer 0). No
  activation without a surface to watch it on.
- **Failure conditions:** any protected-field change, any cross-tenant read, any gateway call outside the authorized
  node, any mutation of an artifact -> immediate abort.
- **Rollback boundary:** the smallest reversible unit. For the chain, that is the single feature flag; flipping it
  off returns the system to safe-by-default with no residue (mutation is structurally impossible, so there is
  nothing to undo beyond audit rows, which are inert).
- **Disable mechanism:** a single config flag set to false (once Layer 1 exists), verified by the Layer-0 surface
  reporting `SafeByDefault`. Until Layer 1 exists, the only disable mechanism is a code revert -- which is itself a
  reason not to activate yet.

## E. recommended activation order (smallest, lowest-risk, read-only first)

1. **Layer 0 -- Observability (read-only).** Add a dev-gated, read-only endpoint that surfaces the already-computed
   `ProbeRefreshChainDiagnostics` / `ProtocolStationDiagnostics` / `ProtocolDiagnosticsRollup` (safe-by-default
   posture, station maturity, chain switch state). Executes nothing. *Unlocks:* the operator visibility that every
   later activation's invariants require. Highest learning value at zero runtime risk.
2. **Layer 1 -- Config-bound reversible flag (no behavior change).** Bind `ChainAssemblyOptions` /
   `ProbeRefreshExecutorOptions` from `IConfiguration`, defaults false. *Unlocks:* a reviewable, reversible,
   measurable switch (the invariant in D) instead of a code edit. Still no execution.
3. **Layer 2 -- Dev-only, audit-only, non-mutating probe run.** In dev, enable the chain + gateway execution +
   executor + audit persistence for ONE niche/tool. Produces a real gateway fetch + one audit ledger row; mutates no
   artifact (impossible). *Unlocks:* the first end-to-end dormant-chain evidence, fully rollback-able by the Layer-1
   flag and measurable via the Layer-0 surface.

Do not enable more than one component per step; each step proves its layer (and is observable on the prior layer's
surface) before the next. **Stop at Layer 2.** Layer 3 (merge writer) is deferred to a future slice and is blocked
by Evidence Readiness Gate 4/5.

## F. factory maturity assessment

| dimension | maturity | note |
|---|---|---|
| Protocol definition | **Mature** | 15 validated station cards; startup fail-fast |
| Runtime orchestration | **Partial** | ProtocolNodeRunner exists (probe-only, no caller); stations driven by sentinels, not ids |
| Tool execution | **Live / Partial** | authz + audit enforced live; rate-limit/timeout/idempotency/tenant-tier absent |
| Evidence governance | **Strong** | Evidence Readiness Gates v1; integrity guard; calibration baseline |
| Decision integrity | **Strong** | settlement integrity guard; mismatch remediation; 59-run integrity-clean corpus |
| Observability | **Weak (runtime)** | diagnostics computed but surfaced nowhere; telemetry log-only; no metrics |
| Rollback discipline | **Strong by design, untested in activation** | safe-by-default flags; mutation structurally impossible; 4-layer protected-field boundary -- but no flag has ever been flipped in a running system and none is config-bound |

- **Greatest strengths:** exceptional safe-by-default dormancy and the four-layer protected-field boundary; a live
  Tool Gateway with fail-closed authorization; strong evidence/decision-integrity governance.
- **Greatest architectural risks:** (1) the **observability gap** -- you cannot measure an activation you cannot
  see; (2) the **config-binding gap** -- activation is a code edit, not a reversible flag, so it is neither
  operator-reviewable nor cleanly disable-able; (3) **maturity-as-permission** -- reading `RuntimeMaturity.Mature`
  or "DI-wired" as "ready to activate".
- **Highest-leverage next investment:** the Layer-0 observability surface + Layer-1 config-bound flags. Together they
  make every future activation reversible AND measurable -- the precondition the deferred-runtime ledger names for
  executor/chain activation, and the cheapest way to convert dormant scaffolding into a controllable runtime.

## answer to the readiness question

**Is the Cognitive Factory ready for its first controlled runtime activation?**

- For the **Tool Gateway**: it is already activated for the current pipeline (authz + audit). No further activation
  is needed to keep running; the cloud-runtime guarantees (rate-limit/timeout/idempotency/tenant-tier) are the
  partial-readiness gap.
- For the **dormant Protocol/Probe-Refresh runtime**: **NO**, not for a mutating or decision-affecting activation.
- The **single missing capability that blocks safe activation** is a **config-bound, reversible feature flag paired
  with a runtime observability surface**. Until a dormant component can be turned on by a reviewable switch and
  watched on a diagnostics surface, no activation is measurable or cleanly reversible -- so the correct first
  activation is the read-only observability surface itself (Part E, Layer 0), which executes nothing.

## what did not change

No runtime behavior, no feature flag, no DI wiring, no execution order, no Tool Gateway behavior, no Probe Refresh
enablement, no ProtocolNodeRunner change, no cognitive protocol, no prompt, no settlement, no calibration, no buyer
behavior. Read-only assessment; the only artifact produced is this document (+ a current-slice status line).

## related docs

- `02 Platform/architecture/governance/evidence-readiness-gates-v1.md` -- mutating runtime activation is a Gate 4/5
  activity; this assessment applies that sequence to the runtime.
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- the canonical deferral
  list whose executor/chain preconditions (entries 2, 13) this assessment verifies against code.
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`,
  `protocol-station-runtime-adoption-readiness-v1.md`, `factory-line-balance-v1.md` -- prior per-subsystem readiness;
  this doc unifies them against current code and adds the cross-component activation order.

## recommended next slice

**Cognitive Factory Observability Surface v1** (implementation): add the dev-gated, read-only diagnostics endpoint
(Part E, Layer 0). It executes nothing, surfaces the safe-by-default posture, and is the prerequisite for any later
activation. Defer config-bound flags (Layer 1) and any execution (Layer 2) to subsequent slices, each gated by this
doc's invariants.
