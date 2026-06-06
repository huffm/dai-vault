# Deferred Runtime Decisions Ledger v1

**date:** 2026-06-05
**status:** operational tracking doc. not doctrine. one row per consciously deferred runtime decision so deferral stays a choice, not an accident.
**scope:** the probe-refresh chain and adjacent runtime/cloud/economic boundaries. each entry records what was deferred, why, what would trigger revisiting it, the slice that should pick it up, the risk if it is forgotten, and current status. update the Status line when a slice changes the answer; do not delete rows, mark them Resolved with the resolving slice.

## How to use this

- A deferral is a decision. If you are about to "just wire it up", check here first -- the current choice and its reason may still hold.
- When a Revisit trigger fires, open the Proposed future slice (or write a new one) and update Status.
- Keep entries operational. Architecture rationale lives in the linked docs; this is the index of open deferrals.
- Status vocabulary: `Deferred` (decided, not yet revisited), `Deferred -- seam shipped, dormant` (the abstraction exists but is inert), `In review` (a slice is actively reconsidering), `Resolved` (a slice changed the choice; name it).

## Entries

### 1. Interrogate requests; the orchestrator (not Interrogate) triggers any Perceive refresh
- **Decision:** who is allowed to trigger a targeted Perceive refresh from a probe gap.
- **Current choice:** `interrogate.probe` emits a structured `ProbeRequest` only. It does not fetch, does not call the Tool Gateway, and does not trigger Perceive. A separate platform orchestrator (Decision -> Authorization -> Executor) decides. Cognitive stations never gain retrieve-tool power.
- **Why deferred:** keeps the cognitive/plumbing boundary clean; lets the chain be built and tested dormant before any loop is closed.
- **Revisit trigger:** a real product need for a within-run refresh loop, with calibration evidence that refresh improves outcomes.
- **Proposed future slice:** Probe Refresh Loop Activation v1 (orchestrator-driven, flagged).
- **Risk if forgotten:** someone wires `interrogate.probe` directly to a retrieve tool, collapsing the boundary and widening the gateway permission matrix.
- **Status:** Deferred -- seam shipped, dormant. See `protocol-node-specs.md` (Probe), probe-refresh chain.

### 2. ProbeRefreshExecutor activation (default-disabled flag)
- **Decision:** whether the executor may actually call the Tool Gateway in any environment.
- **Current choice:** `ProbeRefreshExecutorOptions.Enabled = false` by default; production DI registers it disabled. It is the only chain seam that can touch the gateway, and only at `platform.retrieve`, only after Authorization says Authorized.
- **Why deferred:** no merge/observability story yet; enabling it would make real external fetches with nowhere for the result to go.
- **Revisit trigger:** the merge writer and its observability land, plus a per-tenant enablement need.
- **Proposed future slice:** wire `ProbeRefreshExecutorOptions` to config (appsettings) behind a per-tenant flag, still default disabled.
- **Risk if forgotten:** a caller flips the flag in production before there is a place to use fetched context, causing unaccounted external calls.
- **Status:** Deferred -- seam shipped, dormant.

### 3. Probe-refresh artifact mutation / merge writer
- **Decision:** whether fetched/derived refresh context is ever written into the persisted artifact.
- **Current choice:** nothing mutates the artifact. The chain produces derived views, a merge plan, a review, a dry-run projection, and an audit record only. No writer exists.
- **Why deferred:** artifact mutation is irreversible-ish and high blast-radius; it must be flagged, audited, and rollback-able first.
- **Revisit trigger:** an audit record reaches `ReadyForPersistCandidate` and a product decision to actually merge.
- **Proposed future slice:** Probe Refresh Merge Writer v1 (flagged, rollback from the before-payload reference, dormant first).
- **Risk if forgotten:** the dormant chain is mistaken for live behavior, or a writer is added without the audit/rollback guarantees the chain was designed around.
- **Status:** Deferred -- planning/dry-run/audit seams shipped, no writer.

### 4. Confidence / posture / lean mutation
- **Decision:** whether any probe-refresh path may change confidence, posture, or lean.
- **Current choice:** forbidden. The merge planner classifies confidence/posture/lean/tenant/run fields as protected; proposing them blocks the audit envelope. Confidence stays deterministic (`SportsEvaluator`); posture stays the clamped enum.
- **Why deferred:** these are the calibrated outputs; moving them needs reconciled-outcome evidence, not a refresh signal.
- **Revisit trigger:** calibration evidence (see entry 12) plus an explicit doctrine change.
- **Proposed future slice:** none until calibration justifies it; would be its own scoped slice with its own review.
- **Risk if forgotten:** a "harmless" refresh quietly shifts confidence/posture and corrupts calibration and the product's trust model.
- **Status:** Deferred -- protected by the merge planner's forbidden-field guard.

### 5. Decide recommendation vs actual Decide mutation
- **Decision:** whether the probe-refresh Decide step changes the decision or only recommends.
- **Current choice:** `ProbeRefreshDecideRecommendation` produces a conservative recommendation against explicit current-read context only. It mutates no posture/confidence/lean and writes nothing.
- **Why deferred:** changing the decision in-flight needs the merge writer (entry 3) and the confidence/posture decision (entry 4) settled first.
- **Revisit trigger:** merge writer exists and a product need to act on a recommendation within a run.
- **Proposed future slice:** Probe Refresh Decide Application v1 (after the merge writer), flagged.
- **Risk if forgotten:** a recommendation is treated as an applied decision without the guardrails that separate the two.
- **Status:** Deferred -- recommendation seam shipped, dormant.

### 6. Synthesize preview vs user-facing final artifact
- **Decision:** whether probe-refresh Synthesize output reaches the user.
- **Current choice:** `ProbeRefreshSynthesizePreview` produces preview-only presentation text. It does not touch the synthesize protocol, the final artifact, or anything user-facing.
- **Why deferred:** user-facing output requires the merge writer and a UI/contract decision; preview proves the shape without exposure.
- **Revisit trigger:** a product decision to surface refreshed context to users, with the merge writer in place.
- **Proposed future slice:** Probe Refresh Synthesize Surfacing v1 (after merge writer + contract review).
- **Risk if forgotten:** preview text is mistaken for shippable copy and leaks into a user surface unreviewed.
- **Status:** Deferred -- preview seam shipped, dormant.

### 7. Audit persistence vs artifact persistence
- **Decision:** what the audit store is allowed to persist.
- **Current choice:** the audit store persists audit ledger rows only (idempotent by key). It never writes artifacts or AgentRun rows. The read surface is read-only and tenant-scoped.
- **Why deferred:** audit evidence must exist and be trusted before any artifact write path is built (entry 3).
- **Revisit trigger:** the merge writer slice; it will read `ReadyForPersistCandidate` audit rows.
- **Proposed future slice:** Probe Refresh Merge Writer v1 consumes audit rows; no change to the store/read surface themselves expected.
- **Risk if forgotten:** the audit store is later overloaded to also write artifacts, blurring evidence vs effect.
- **Status:** Deferred -- audit store + read surface shipped; artifact persistence not built.

### 8. Dev-only audit read endpoint
- **Decision:** whether to expose the audit read surface over HTTP.
- **Current choice:** service-level read only. No HTTP endpoint. A dev-only gated endpoint was considered and declined.
- **Why deferred:** unnecessary to prove the read surface; an endpoint widens the tenant-isolation surface that must be guarded.
- **Revisit trigger:** an operator/support need to inspect audit history outside tests.
- **Proposed future slice:** Probe Refresh Audit Read Endpoint v1 -- env-gated (development/admin), non-public, tenant from caller identity not a query parameter.
- **Risk if forgotten:** a future endpoint is added without env-gating or tenant-from-identity, leaking cross-tenant audit data.
- **Status:** Deferred -- read service shipped, no endpoint.

### 9. pgvector / memory-backed probe
- **Decision:** whether `interrogate.probe` consults governed vector memory.
- **Current choice:** deterministic probe only, built from signal follow-up records. No pgvector, no memory tool. `allowed_memory_queries` is empty on every station card.
- **Why deferred:** memory is offline-first per the cloud plan; it must be provisioned, ingested, and reviewed before any synchronous station opens against it.
- **Revisit trigger:** pgvector provisioned offline-first, calibration-side ingestion built, and a calibration finding that supports opening `memory.prior_runs.search` / `memory.signal_failures` to `interrogate.probe`.
- **Proposed future slice:** Governed Memory Probe Augmentation v1 (per-tenant flag, async, via the gateway).
- **Risk if forgotten:** memory is wired synchronously into a run without the offline-first/governance discipline, or as an ungoverned context dump.
- **Status:** Deferred -- specified in `protocol-station-blueprint-v1.md` section 10, not built.

### 10. Kubernetes / AKS / Azure Functions deployment
- **Decision:** the runtime deployment target beyond the current launch shape.
- **Current choice:** sports v1 targets Azure Container Apps; AKS/Kubernetes, multi-region, and Azure Functions tool transports are deferred per the cloud plan.
- **Why deferred:** launch does not need them; they are heavier abstractions than warranted now.
- **Revisit trigger:** scale, multi-region, or a tool that genuinely needs the Functions transport.
- **Proposed future slice:** Cloud Runtime Expansion v1 (separate from cognitive work).
- **Risk if forgotten:** premature infra work, or a tool added assuming a transport the platform does not run.
- **Status:** Deferred -- see `cloud-tool-runtime-plan.md`.

### 11. Tenant / Stripe / economic boundary integration
- **Decision:** how probe-refresh cost and enablement bind to the tenant/billing boundary ("stripe = truth").
- **Current choice:** the chain carries `TenantKey` through every seam and audit row and is tenant-scoped on read, but there is no cost-class enforcement, tenant-tier gating, or Stripe-driven enablement wired to it.
- **Why deferred:** the chain is dormant and free today; economic gating matters only once refresh actually fetches/merges (entries 2, 3).
- **Revisit trigger:** executor activation or merge writer, where real cost and per-tenant entitlement begin to apply.
- **Proposed future slice:** Probe Refresh Tenant Entitlement v1 (cost class + tenant-tier gate at activation).
- **Risk if forgotten:** refresh is enabled without a billing/entitlement gate, decoupling spend from the tenant economic boundary.
- **Status:** Deferred -- tenant key threaded, no economic gate.

### 12. Calibration proof before posture-threshold changes
- **Decision:** what evidence is required before any confidence/posture threshold moves.
- **Current choice:** thresholds are manually calibrated; a handful of leans is too small to move a threshold. Gates only flag; they do not auto-adjust confidence.
- **Why deferred:** not enough reconciled game-day outcomes exist to move a threshold responsibly.
- **Revisit trigger:** enough reconciled `AgentRunOutcome` -> `AgentRunEvaluation` evidence to support a specific threshold change.
- **Proposed future slice:** Calibration-Driven Threshold Update v1 (evidence-gated, its own review).
- **Risk if forgotten:** a threshold is changed on intuition or on a refresh signal, undermining calibration credibility.
- **Status:** Deferred -- standing decision recorded in handoffs; gates flag only.

### 13. Probe-refresh chain assembly activation
- **Decision:** whether the assembled probe-refresh chain runs in production.
- **Current choice:** the chain is assembled into one orchestration object (`ProbeRefreshChainAssembly`) but is disabled by default (options `Enabled = false`), makes no gateway call unless `AllowGatewayExecution = true` and the executor is enabled, persists only the audit ledger row when `PersistAuditRecord = true`, mutates no artifact, and is wired into no pipeline and no endpoint.
- **Why deferred:** activation depends on the still-open upstream deferrals -- executor activation (entry 2), the merge writer (entry 3), and the orchestrator trigger (entry 1) -- and on a product decision to run it.
- **Revisit trigger:** entries 1-3 resolve and there is a real need to run the chain within or after a run.
- **Proposed future slice:** Probe Refresh Chain Activation v1 (orchestrator-driven, flagged, after the merge writer).
- **Risk if forgotten:** the dormant assembly is mistaken for live behavior, or is enabled before the writer/observability/trigger story exists.
- **Status:** Deferred -- assembly, diagnostics, and Activation Readiness Review v1 shipped, dormant. Activation still waits on entries 1-3, feature-flag/config design, telemetry emission, operator review, tenant/economic gating, and product approval.

### 14. Genericization of probe-refresh-specific seams (intake / re-weigh / recommendation logic)
- **Decision:** whether the probe-refresh Perceive intake, Discern re-weigh, and Decide recommendation seams are generalized into platform-level station logic now, or kept probe-refresh specific.
- **Current choice:** kept probe-refresh specific. `ProbeRefreshPerceiveIntake` is hard-keyed to sports `ToolIds.*` and competition context types; `ProbeRefreshDiscernReweigh` and `ProbeRefreshDecideRecommendation` are single-consumer. Only the result *shape* (`Status` + `Reason` + `ErrorMessage?` + `IsX`, plus the `(Step, Reached, Outcome)` trace) is past the rule-of-three and approved for harvest; the station *logic* is not.
- **Why deferred:** generalizing the logic on one consumer would be a speculative one-use abstraction, the opposite of the discipline the rest of the probe-refresh chain respected. The shared result envelope must land first so a future second consumer has a contract to adopt.
- **Revisit trigger:** the Generic Station Result Envelope v1 slice lands AND a second, non-probe consumer of signal intake / re-weigh / recommendation appears.
- **Proposed future slice:** Generic Station Result Envelope v1 (the result-shape harvest, recommended next); later a `PerceiveSignalIntake` / generic Decide policy guard only once a second use case exists.
- **Risk if forgotten:** a second consumer copies `ProbeRefreshPerceiveIntake` wholesale (forking sports payload logic into the platform), or the probe-specific logic is prematurely lifted into shared infrastructure and ossifies on one niche.
- **Status:** Deferred -- generic result/trace shape shipped, dormant. Recorded by Factory Line Balance Review v1 (2026-06-06). Progressed by Generic Station Result Envelope v1 (2026-06-06): the shared `ProtocolResultEnvelope<TStatus>` and `ProtocolStepTrace` contracts landed in DevCore.Api.Protocols, and the chain step trace was re-expressed against the trace (diagnostics + assembly accessor). The 15 seam result records were NOT retrofitted and the probe-specific intake/re-weigh/recommendation *logic* was NOT generalized -- broader genericization stays deferred until a second, non-probe consumer exists. Progressed again by Probe Refresh Merge Review Envelope Adoption v1 (2026-06-06): the first real seam adopted the envelope -- `ProbeRefreshMergeReviewResult.ToEnvelope()` projects the domain status to a `ProtocolResultDisposition` while keeping every domain field (MayMerge, RequiresManualReview, BlockedReasons, TelemetryEvent) intact. This is one adopted seam, not a migration: the other 14 seam result records still keep their own shapes, and the probe-specific station logic is still not generalized. See `factory-line-balance-review-v1.md`.

## Maintenance

- When a slice resolves an entry, set Status to `Resolved (<slice name>, <date>)` and keep the row.
- When a new deferral is made in a handoff addendum, add a row here in the same slice so the two stay in sync.
- This ledger does not replace the handoff log; it is the durable index of open deferrals the handoff entries create.

## References

- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md` (station boundaries, memory section)
- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md` (Probe: requests, does not retrieve)
- `02 Platform/architecture/cloud-tool-runtime-plan.md` (Tool Gateway, Functions, pgvector deferrals)
- `06 Execution/handoffs/current-slice.md` (the probe-refresh chain addenda this ledger indexes)
