# Protocol Station Runtime Adoption Readiness v1

**date:** 2026-06-07
**status:** readiness review. docs only. no station adoption, activation, execution, artifact mutation, endpoint, prompt change, Tool Gateway behavior change, or schema change.
**scope:** evaluate whether any cognitive protocol station should receive a read-only runtime caller next, and define the gates before any future activation.

## Executive summary

No station should be activated by default from the current state.

The station-card contract is materially stronger than it was during Factory Line Balance v1: all 15 registry stations now carry ownership, maturity, artifact mutation policy, input/output metadata, forbidden behavior, fallback behavior, and status semantics. Diagnostics and the rollup can inspect that metadata read-only. That is enough to make the station line legible; it is not enough to justify runtime adoption.

The deciding gap is not technical shape. It is the absence of a concrete read-only caller with a product or operator need. Activating a runner simply because the station cards are complete would convert governance metadata into behavior without a reason. The safest technical candidate, if tomorrow's deliberation demands a caller, is Synthesize runtime inspection because the Synthesize trio is platform-owned, deterministic, no-tool, and no-model. Even there, the first slice should inspect existing outputs only and must not mutate artifacts or change delivery.

Primary recommendation: **No Runtime Adoption Yet / Product Deliberation v1**. Decide whether the next priority is factory runtime maturity or sports product polish before adding any caller.

Backup implementation slice if a read-only caller is explicitly approved: **Synthesize Runtime Inspection v1**.

## Current station contract state

The runtime contract now reflects the core station blueprint in code:

- `ProtocolStationCard` declares station identity, macro protocol, micro-action, purpose, allowed tools, allowed model-call policy, cost class, telemetry tags, quality gates, calibration hooks, owner, runtime maturity, artifact mutation policy, artifact fields read/written, input contract, output contract, allowed scripts/reflexes, allowed memory queries, fallback behavior, forbidden behavior, status semantics, and optional token budget.
- `ProtocolRegistry.Default()` populates the 15 canonical station cards from code: Perceive, Interrogate, Discern, Decide, and Synthesize.
- `ProtocolRegistryValidator` checks completeness and tool-card alignment.
- `ProtocolStationDiagnostics` exposes per-station metadata without executing stations.
- `ProtocolDiagnosticsRollup` summarizes owner, maturity, runner support, and status-semantics completeness without executing stations.

The current live runtime is still the existing pipeline:

- Eleven cognitive station outputs are model-emitted inside one shared analyze call.
- `interrogate.probe` is deterministic platform output from `CognitiveProtocolBuilder.BuildProbe`.
- Synthesize is deterministic platform composition/delivery.
- `ProtocolNodeRunner.ExecuteAsync` supports `interrogate.probe` only.
- Perceive signal intake, Perceive observation collection, and Discern station runner contracts are dormant/read-only groundwork, not production routing.

## Current station status semantics state

All 15 station cards carry status semantics:

- uncertainty behavior
- skip behavior
- blocked behavior
- failure behavior
- not-applicable behavior
- readiness interpretation

The semantics are metadata and diagnostics only. They do not authorize execution, model-call splitting, Tool Gateway access, artifact mutation, confidence/posture/lean mutation, or endpoint exposure.

Observed status model from code and vault review:

- Model-owned stations generally defer uncertainty to bounded model output or surface it diagnostically.
- `interrogate.probe` can request follow-up context as a station status meaning, but it must not retrieve or call tools.
- Platform-owned deterministic stations either preserve existing behavior, skip safely, or fail closed depending on station role.
- Decide stations explicitly do not authorize confidence, posture, or lean mutation.
- Synthesize stations mark uncertainty as not applicable, preserve existing behavior for skip/blocked cases, and surface failure diagnostics.

## Runtime adoption candidates

Candidate ranking is based on current factory bottleneck, not on what was most recently built.

1. **No Runtime Adoption Yet / Product Deliberation v1** - preferred. No concrete read-only caller is currently identified. The next decision should clarify whether DAI needs factory runtime maturity or sports product polish next.
2. **Synthesize Runtime Inspection v1** - safest backup if a caller is approved. Synthesize is deterministic, platform-owned, no-tool, no-model, and already active. The slice would inspect existing outputs only.
3. **Protocol Station Runtime Readiness Diagnostics v1** - valid if the team wants more machine-readable readiness reporting before a caller. This is lower priority than product/factory deliberation because rollup diagnostics already exist.
4. **Perceive Signal Intake Read Model v1** - technically plausible but premature until a product, operator, or analyzer-read caller actually needs normalized Perceive observations.
5. **Interrogate.Probe runtime adoption expansion** - not recommended. Probe is mature, but further adoption risks collapsing the no-direct-retrieve boundary.
6. **Discern/Decide runner adoption** - not recommended. These stations influence judgment, confidence, posture, lean, and final position; they need broader guardrails and calibration policy first.

## Station-by-station readiness matrix

| Station id | Macro | Owner | Runtime maturity | Mutation policy | Status semantics complete? | Runner support today? | Tool access needed? | Read-only caller possible? | Activation risk | Recommendation |
|---|---|---|---|---|---|---|---|---|---|---|
| `perceive.detect` | Perceive | Model | Partial | ScopedOutputOnly | Yes | No | Shared analyze only; no direct station tool | Yes, via Perceive read model | Medium | Keep projection/read-only only until a concrete consumer exists |
| `perceive.frame` | Perceive | Model | Partial | ScopedOutputOnly | Yes | No | Shared analyze only; no direct station tool | Yes, via Perceive read model | Medium | Keep projection/read-only only |
| `perceive.aim` | Perceive | Model | Partial | ScopedOutputOnly | Yes | No | Shared analyze only; no direct station tool | Yes, via Perceive read model | Medium | Keep projection/read-only only |
| `interrogate.question` | Interrogate | Model | Partial | ScopedOutputOnly | Yes | No | Shared analyze only; no direct station tool | Limited; mostly diagnostic | Medium | Keep model-owned only for now |
| `interrogate.probe` | Interrogate | Platform | Mature | ScopedOutputOnly | Yes | Yes | No direct tools; retrieve remains platform plumbing | Yes, inspection-only | High | Do not expand runtime adoption; no direct retrieve, no Perceive self-invocation |
| `interrogate.verify` | Interrogate | Model | Partial | ScopedOutputOnly | Yes | No | Shared analyze only; no direct station tool | Limited; mostly diagnostic | Medium | Keep model-owned only for now |
| `discern.weigh` | Discern | Hybrid | Partial | ScopedOutputOnly | Yes | No dedicated production runner; dormant groundwork only | Shared analyze plus deterministic signal grading | Yes, inspection-only | High | Do not adopt until generic Discern guardrails and caller exist |
| `discern.contrast` | Discern | Model | Partial | ScopedOutputOnly | Yes | No | Shared analyze only; no direct station tool | Limited; mostly diagnostic | High | Keep model-owned only for now |
| `discern.stress` | Discern | Model | Partial | ScopedOutputOnly | Yes | No | Shared analyze only; no direct station tool | Limited; mostly diagnostic | High | Keep model-owned only for now |
| `decide.resolve` | Decide | Hybrid | Partial | ScopedOutputOnly | Yes | No | Shared analyze plus deterministic lean clamp | Limited; high governance burden | High | Do not adopt; no lean mutation |
| `decide.position` | Decide | Hybrid | Partial | ScopedOutputOnly | Yes | No | Shared analyze plus deterministic posture clamp | Limited; high governance burden | High | Do not adopt; no posture mutation |
| `decide.justify` | Decide | Hybrid | Partial | ScopedOutputOnly | Yes | No | Shared analyze plus deterministic confidence owner | Limited; high governance burden | High | Do not adopt; no confidence mutation |
| `synthesize.integrate` | Synthesize | Platform | Mature | ScopedOutputOnly | Yes | No station runner; deterministic code exists | No | Yes | Low | Safest backup read-only inspection candidate |
| `synthesize.compose` | Synthesize | Platform | Mature | InitialArtifactAssembly | Yes | No station runner; deterministic code exists | No | Yes | Medium | Inspect only; do not confuse initial assembly with post-hoc mutation |
| `synthesize.deliver` | Synthesize | Platform | Mature | InitialArtifactPersistence | Yes | No station runner; deterministic code exists | No | Yes | Medium | Inspect only; no endpoint or delivery behavior change |

No code/vault discrepancy was found in the canonical station list. The code registry uses the same 15 station ids expected by the vault docs.

## Required activation gates

Any future station runtime adoption, even read-only, must have:

- explicit feature flag, default off.
- dev/local-only first.
- named caller and reason for the caller.
- tenant/run/correlation boundary if data is involved.
- diagnostics captured before and after execution/inspection.
- no artifact mutation by default.
- no confidence, posture, or lean mutation.
- no Tool Gateway call unless the station card and Tool Gateway policy both allow it.
- no model call unless the station card permits it and the slice explicitly approves the call count.
- audit and rollback design before any future mutation.
- calibration proof before confidence/posture/lean changes.
- clear operator review path before user-facing behavior changes.
- registry validation and diagnostics rollup passing before activation.
- tests that prove disabled-by-default behavior.
- a dedicated implementation slice; this review does not approve activation.

## Required diagnostics before adoption

Minimum diagnostic surface before a read-only caller:

- per-station snapshot from `ProtocolStationDiagnostics`.
- rollup snapshot from `ProtocolDiagnosticsRollup`.
- station-card validation findings, including missing metadata/status semantics.
- runner support vs unsupported-station status.
- owner and runtime maturity.
- mutation policy and protected-field status.
- allowed tools, allowed model-call policy, and policy check result.
- status semantics and readiness interpretation.
- before/after artifact diff proving no mutation for read-only callers.
- tenant/run/correlation metadata on any diagnostic record that touches run data.
- explicit warning when a station remains shared-analyze output rather than a real station runner.

## Required tests before adoption

Any runtime adoption slice needs tests for:

- feature flag default off.
- no execution when disabled.
- no artifact mutation in read-only mode.
- no confidence, posture, or lean mutation.
- no station id changes.
- allowed tools unchanged unless the slice explicitly changes them.
- default registry validation clean.
- station diagnostics and rollup expose the adopted station state.
- unsupported stations still fail closed or report unsupported.
- no Tool Gateway call unless explicitly expected and policy-authorized.
- no model call count increase unless explicitly approved.
- tenant/run/correlation propagation when run data is inspected.
- exact before/after artifact equality for read-only paths.
- operator/diagnostic trace includes station id, status, reason, and error where applicable.

## Blocked or forbidden moves

These remain blocked:

- direct Interrogate -> Perceive self-invocation.
- direct tool power on `interrogate.probe`.
- production activation without tenant/auth boundary.
- artifact mutation without audit trail.
- confidence/posture/lean changes without calibration proof.
- merge writer before explicit approval.
- endpoint before explicit gate.
- Tool Gateway behavior change from a station adoption slice.
- model-call split from a station adoption slice.
- FastAPI prompt change.
- Angular change.
- DB/schema migration.
- using Synthesize inspection as a back door for artifact merge/write behavior.
- treating station-card metadata or status semantics as runtime permission.

## Deliberation questions for 2026-06-08

- Is the next priority factory runtime maturity or sports product polish?
- Is there a real read-only station consumer now, or should station work stay in docs/diagnostics?
- Should Synthesize be the first runtime-adoption target if a caller is required?
- Should Perceive intake remain projection-only until a product need appears?
- Should Discern and Decide wait until generic guardrails and calibration policy are stronger?
- What minimum product-facing artifact improvement would validate the platform better than more runtime groundwork?
- What operator or developer surface actually needs station-runtime inspection?
- Would a no-runtime product deliberation slice produce a sharper next implementation target?

## Recommended next implementation slice

**Primary: No Runtime Adoption Yet / Product Deliberation v1.**

Why: The current bottleneck is decision clarity, not another dormant caller. Station contracts and status semantics are now strong enough to support future adoption, but no concrete read-only consumer has been identified. The team should choose whether the next tranche optimizes platform runtime maturity, sports artifact product value, or calibration feedback before adding another runtime seam.

Expected output: one vault-first decision note that picks the next product/platform axis and names a concrete implementation slice. It should not change runtime code.

## Backup next slice

**Synthesize Runtime Inspection v1.**

Use this only if the deliberation identifies a real read-only caller. It should inspect the existing Synthesize trio outputs and station card metadata, expose no endpoint by default, call no model/tool, mutate no artifact, and leave delivery behavior unchanged.

Synthesize is the safest technical target because it is platform-owned, mature, deterministic, no-tool, no-model, and already part of normal artifact assembly. Its main risk is conceptual: confusing initial compose/deliver responsibilities with post-hoc artifact mutation. That risk is manageable only if the slice is inspection-only.

## Anti-goals

- Do not activate `DiscernStationRunner`.
- Do not route production Perceive intake through the collector.
- Do not route analyzer seed output through a new production caller.
- Do not deepen probe-refresh activation.
- Do not add a merge writer.
- Do not add an endpoint.
- Do not split model calls.
- Do not change FastAPI prompts.
- Do not change Angular.
- Do not change Tool Gateway execution behavior.
- Do not mutate artifacts.
- Do not change confidence/posture/lean behavior.
- Do not add schema migrations.
- Do not add MCP, pgvector, Azure Functions, Kubernetes, tenant/Stripe, or production secret changes.

## References reviewed

- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolStationCard.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolRegistry.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolRegistryValidator.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolStationDiagnostics.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolDiagnosticsRollup.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProtocolNodeRunner.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/PerceiveSignalIntake.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/PerceiveSignalObservationCollector.cs`
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainAssembly.cs`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/factory-line-balance-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/factory-line-balance-review-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_VAULT_ROOT>/06 Execution/handoffs/current-slice.md`

## Verification performed for this note

- docs-only change.
- no runtime files changed.
- no schema/migration files changed.
- staged `git diff --check` passed.
- staged exact local path scan found no exact local machine paths.
- ASCII check for this readiness note passed.
