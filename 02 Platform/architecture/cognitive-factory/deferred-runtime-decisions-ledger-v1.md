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
- **Current choice:** kept probe-refresh specific for station logic. `ProbeRefreshPerceiveIntake` still owns the sports `ToolIds.*` switch and context summarizers; `ProbeRefreshDiscernReweigh` and `ProbeRefreshDecideRecommendation` are still single-consumer. Shared contract infrastructure now exists around the seam: `ProtocolResultEnvelope<TStatus>`, `ProtocolStepTrace`, and the dormant generic `PerceiveSignalIntake` observation/result contract. None of those replace probe-refresh logic or wire into production.
- **Why deferred:** generalizing the logic on one consumer would be a speculative one-use abstraction, the opposite of the discipline the rest of the probe-refresh chain respected. The shared result envelope must land first so a future second consumer has a contract to adopt.
- **Revisit trigger:** a second, non-probe consumer of re-weigh / recommendation logic appears, or the generic Perceive signal intake adoption slice needs to remove duplicated intake mapping without broad probe-refresh refactoring.
- **Proposed future slice:** keep genericization incremental. Perceive signal intake adoption is tracked separately in entry 15; Discern re-weigh and Decide recommendation logic remain probe-refresh specific until a second consumer exists.
- **Risk if forgotten:** a second consumer copies `ProbeRefreshPerceiveIntake` wholesale (forking sports payload logic into the platform), or the probe-specific logic is prematurely lifted into shared infrastructure and ossifies on one niche.
- **Status:** Deferred -- generic result/trace shape shipped, dormant. Recorded by Factory Line Balance Review v1 (2026-06-06). Progressed by Generic Station Result Envelope v1 (2026-06-06): the shared `ProtocolResultEnvelope<TStatus>` and `ProtocolStepTrace` contracts landed in DevCore.Api.Protocols, and the chain step trace was re-expressed against the trace (diagnostics + assembly accessor). The 15 seam result records were NOT retrofitted and the probe-specific intake/re-weigh/recommendation *logic* was NOT generalized -- broader genericization stays deferred until a second, non-probe consumer exists. Progressed again by Probe Refresh Merge Review Envelope Adoption v1 (2026-06-06): the first real seam adopted the envelope -- `ProbeRefreshMergeReviewResult.ToEnvelope()` projects the domain status to a `ProtocolResultDisposition` while keeping every domain field (MayMerge, RequiresManualReview, BlockedReasons, TelemetryEvent) intact. Progressed again by Perceive Signal Intake Standardization v1 (2026-06-06): a generic `PerceiveSignalIntake` / `PerceiveSignalObservation` contract and one-way `PerceiveRefreshView` projection landed, but the probe-refresh intake result remains in place and no production caller consumes the generic projection. See entry 15 for adoption.

### 15. Perceive signal intake adoption across analyzer seed and refreshed context
- **Decision:** when the generic Perceive signal intake contract becomes an adopted production intake path for analyzer seed signals, platform-provided refresh observations, manual inputs, and future memory-backed observations.
- **Current choice:** projection/collection support only. `PerceiveSignalIntake`, `PerceiveSignalObservation`, `PerceiveSignalContext`, `PerceiveSignalSource`, `PerceiveSignalOrigin`, and `PerceiveSignalIntakeResult` exist in DevCore.Api.Protocols. One-way projections exist for analyzer seed/model-emitted Perceive fields (`AnalyzerSeedPerceiveSignalProjection`) and probe-refresh/platform-refreshed Perceive views (`ProbeRefreshPerceiveSignalProjection`). `PerceiveSignalObservationCollector` can collect both into one normalized observation set with source/origin counts and rejected-candidate reporting. Analyzer seed parsing, `CognitiveProtocol`, `PerceiveProtocol`, `ProbeRefreshPerceiveIntake`, and the probe-refresh chain are not rewired.
- **Why deferred:** the slice was intentionally behavior-preserving. Adoption would decide where analyzer seed observations are normalized, which production callers consume the generic result, and how refresh observations are threaded without artifact mutation or prompt changes.
- **Revisit trigger:** a production caller needs to consume normalized Perceive signal observations, or a future analyzer-seed routing slice needs run/tenant/correlation/source metadata beyond read-only projection tests.
- **Proposed future slice:** Perceive Signal Intake Runtime Consumer v1 -- route a read-only consumer through the generic contract without model calls, Tool Gateway calls, artifact writes, confidence/posture/lean mutation, endpoints, schema changes, FastAPI prompt changes, or Angular changes.
- **Risk if forgotten:** future loops feed Perceive through inconsistent ad hoc shapes, or probe-refresh remains the only path with a receive seam and the platform copies sports-specific intake logic into wider factory work.
- **Status:** Deferred -- contract/projections/collector shipped, dormant. Created by Perceive Signal Intake Standardization v1 (2026-06-06). Progressed by Perceive Signal Intake Adoption v1 (2026-06-06): analyzer seed/model-emitted Perceive fields (`Detect`, `Frame`, `Aim`) now project into `PerceiveSignalObservation` with field-path signal keys (`perceive.detect`, `perceive.frame`, `perceive.aim`), `SourceKind = AnalyzerSeed`, and `Origin = ModelEmitted`; probe-refresh projections remain `PlatformRefresh` / `ToolFetched`. Progressed again by Perceive Signal Observation Collector v1 (2026-06-06): analyzer seed and platform refresh observations can be collected into `PerceiveSignalObservationSet` with source counts and safe rejection of invalid/no-signal candidates. Production routing remains deferred. Do not mark direct Interrogate -> Perceive refresh, activation, merge writer, artifact mutation, confidence/posture/lean mutation, memory/pgvector, Kubernetes/AKS, tenant/Stripe, or calibration threshold changes resolved from this entry.

### 16. Discern station runner groundwork vs production Discern execution
- **Decision:** when platform-owned Discern station execution becomes an adopted production path for `discern.weigh`, `discern.contrast`, and `discern.stress`.
- **Current choice:** dormant groundwork only. `DiscernStationRunner`, `IDiscernStationRunner`, `DiscernStationExecutionRequest`, `DiscernStationExecutionResult`, `DiscernStationAssessment`, and `DiscernStationAssessmentStatus` exist in DevCore.Api.Protocols. The runner produces deterministic value objects plus `ProtocolResultEnvelope<TStatus>` and `ProtocolStepTrace` projections. It is not DI-registered, not wired into `ProtocolNodeRunner`, and not consumed by analyzer seed, probe-refresh, artifact, endpoint, or merge paths. `ProbeRefreshDiscernReweigh` remains probe-refresh-specific and unchanged.
- **Why deferred:** Discern still needs a real adoption decision: what caller should supply context, whether deterministic assessment should inspect normalized observations, and how to preserve calibrated confidence/posture boundaries. Wiring execution now would imply runtime behavior before the factory has a safe Discern policy.
- **Revisit trigger:** a production or read-only caller needs to inspect Discern station assessment results, or a second non-probe consumer creates reuse pressure for Discern assessment beyond `ProbeRefreshDiscernReweigh`.
- **Proposed future slice:** Discern Station Runner Adoption v1 -- introduce a read-only consumer if there is a concrete caller, still without model calls, Tool Gateway calls, artifact writes, confidence/posture/lean mutation, endpoints, schema changes, FastAPI prompt changes, or Angular changes.
- **Risk if forgotten:** Discern remains model-emitted fields plus probe-refresh-specific re-weighing, so future consumers either copy probe-refresh logic or wire Discern execution directly into production without envelope/trace/status guardrails.
- **Status:** Deferred -- runner groundwork shipped, dormant. Created by Discern Station Runner Groundwork v1 (2026-06-06). Generic Discern re-weigh is not resolved; direct Interrogate -> Perceive refresh, activation, merge writer, artifact mutation, confidence/posture/lean mutation, memory/pgvector, Kubernetes/AKS, tenant/Stripe, and calibration threshold changes remain deferred.

### 17. Station card contract completion before station runtime adoption
- **Decision:** whether any new station runtime adoption should proceed before the station cards encode full ownership boundaries.
- **Current choice:** `Protocol Station Contract Completion v1` progressed the machine-readable contract on 2026-06-06. `ProtocolStationCard` now carries execution owner, runtime maturity, artifact mutation policy, artifact fields read/written, allowed scripts/reflexes, allowed memory queries, input/output contracts, fallback behavior, forbidden behavior, and token budget; all 15 default station cards are populated; validator, station diagnostics, and diagnostics rollup surface the new metadata. `Protocol Station Status Contract v1` progressed it again on 2026-06-07: `ProtocolStationStatusSemantics` now encodes uncertainty, skip, blocked, failure, not-applicable, and readiness interpretation metadata for every station card. Still do not adopt or activate more station runtime behavior yet.
- **Why deferred:** the station contract is metadata-complete for the v1 ownership/status surface, but runtime adoption remains deferred. Per-station model/gateway execution is still not split, no station activation gate has been approved, and future runtime consumers must prove a concrete caller before activation.
- **Revisit trigger:** any proposed runtime consumer for `DiscernStationRunner`, `PerceiveSignalObservationCollector`, memory/document tools, probe-refresh adoption, or per-station execution; or an explicit request to move status semantics from diagnostics metadata into runtime behavior.
- **Proposed future slice:** No Runtime Adoption Yet / Product Deliberation v1 -- decide the next product/platform axis before adding a station caller. Backup only if a concrete read-only caller is approved: Synthesize Runtime Inspection v1.
- **Risk if forgotten:** a future slice may wire station execution, memory, document tools, or artifact writes from a partial manifest and re-create station boundaries in ad hoc code instead of the platform contract.
- **Status:** Deferred -- progressed by Protocol Station Contract Completion v1 (2026-06-06), Protocol Station Status Contract v1 (2026-06-07), and Protocol Station Runtime Adoption Readiness Review v1 (2026-06-07). The readiness review found no concrete read-only caller yet; Synthesize is the safest backup target only if a caller is approved. No station activation, runtime adoption, endpoint, Tool Gateway behavior change, artifact mutation, confidence/posture/lean mutation, schema change, FastAPI prompt change, Angular change, calibration threshold change, memory/pgvector change, Kubernetes/AKS change, tenant/Stripe change, or merge writer.

### 18. Runtime station adoption before a concrete read-only caller exists
- **Decision:** whether station runtime adoption should proceed now that station cards and status semantics are complete.
- **Current choice:** do not adopt or activate any station runtime caller yet. `Protocol Station Runtime Adoption Readiness Review v1` documents the readiness matrix, required activation gates, required diagnostics/tests, blocked moves, and deliberation questions for the 2026-06-08 review. If a caller is approved, the safest backup candidate is Synthesize Runtime Inspection v1 because Synthesize is platform-owned, mature, deterministic, no-tool, and no-model. Even that would be inspection-only.
- **Why deferred:** station cards are now metadata-complete enough to inspect, but no product/operator/developer caller has been proven. Activating a dormant runner or new caller would turn governance metadata into behavior without a concrete need.
- **Revisit trigger:** the 2026-06-08 deliberation names a specific read-only station consumer, or a product-facing artifact improvement requires station-level runtime inspection that cannot be satisfied by existing diagnostics.
- **Proposed future slice:** Sports Brief Signal Table v1. Runtime adoption remains deferred; Synthesize Runtime Inspection v1 remains a backup only after a concrete read-only caller is approved.
- **Risk if forgotten:** future work may activate `DiscernStationRunner`, route Perceive observations, expand `interrogate.probe`, or add Synthesize inspection because the contracts exist, rather than because a bounded caller and activation gate exist.
- **Status:** Deferred -- readiness review shipped, docs only. Product vs Factory Deliberation v1 (2026-06-07) found no concrete read-only runtime caller and recommended product polish next: Sports Brief Signal Table v1. Clarified by Design Readiness and Focus Selection v1 (2026-06-08): the scheduled 2026-06-08 deliberation occurred, named no read-only station consumer, and reaffirmed product polish (primary: Sports Brief Signal Table v1; backup: Sports Artifact Productization Review v1). Runtime station adoption remains Deferred; Synthesize Runtime Inspection v1 stays a runtime backup only if a named caller appears. No station activation, production wiring, endpoint, Tool Gateway behavior change, model-call change, artifact mutation, confidence/posture/lean mutation, schema change, FastAPI prompt change, Angular change, merge writer, memory/pgvector change, Kubernetes/AKS change, tenant/Stripe change, or calibration threshold change.

### 19. Sports signal source and fallback catalog
- **Decision:** whether DAI builds a structured catalog of sports signal sources and fallback ladders (which provider grounds each signal, which proxy substitutes for it, and at what equivalence) as a platform asset.
- **Current choice:** not built. `Sports Brief Signal Table v1` (2026-06-08) surfaces the *current* signal picture for a run -- grounded/weak/missing/unavailable/proxy state, a buyer-facing source-type label, and a fallback NEED -- using only existing artifact fields (`signalAvailability`, `signalFollowUps`, coarse grounded/missing for legacy artifacts). It is a read-only frontend projection. It does not add a source, does not fetch data, and does not encode a reusable source/fallback catalog. The deterministic fallback ladder still lives in `SignalFollowUpEvaluator` (backend) and is not exposed as a standalone catalog.
- **Why deferred:** the table makes the source/fallback gaps visible per run, which is the input a catalog needs. Building the catalog now -- before the table proves which gaps recur and matter to buyers -- would be speculative. No new data source is approved in this slice.
- **Revisit trigger:** the signal table shows a recurring, buyer-relevant gap (for example sharp_public missing or line_movement not implemented across many runs) that justifies a real source or proxy implementation.
- **Proposed future slice:** Sports Signal Source and Fallback Catalog v1 -- a structured source/fallback registry, still gated by real provider availability and cost; or a narrower Line Movement Proxy v1 if that single gap dominates.
- **Risk if forgotten:** the table keeps showing the same unmet fallback needs with no path to close them, or a future slice adds a source ad hoc without a catalog and the source/proxy mapping fragments across the codebase.
- **Status:** Deferred -- created by Sports Brief Signal Table v1 (2026-06-08). Clarified by Sports Artifact Productization Review v1 (2026-06-08): the review confirmed the source/fallback gaps are real and buyer-relevant, and added that a buyer-facing route additionally needs read-only backend metadata -- an availability-fidelity status (to distinguish "not configured / never fetched" from "fetched, empty" so the `Unavailable` state is honest) and a per-signal source-kind (analyzer-seed / platform-derived / tool-fetched) before source provenance can be shown to a buyer. These are read-only metadata enrichments, each its own future slice; they add no provider and activate nothing. Progressed by Buyer Artifact Route v1 (2026-06-08): a read-only buyer-safe signal summary now renders on the analyzer surface (reusing `buildSignalTable` via `buildBuyerSignalSummary` and the existing artifact read endpoint). The buyer summary deliberately collapses `Unavailable` into "Not available" and omits source provenance, because the availability-fidelity status and source-kind metadata named above still do not exist. Progressed by Artifact Copy and Section Order v1 (2026-06-08): buyer copy and reading order were locked (Read Stance moved decision-first; "What Could Change the Read"; state copy "Not available in this run" / "Not clear enough to use" / "Backed by current data"), still collapsing `Unavailable` and omitting source provenance pending the same backend metadata. The deeper relayout that would hoist the Signal Summary above the prose narrative is recorded there as a deferred design decision (it requires breaking the locked two-column answer zone). Source expansion is NOT resolved. Probe-refresh activation is NOT resolved. Confidence/posture/lean mutation is NOT resolved. Tenant/Stripe economic boundary is NOT resolved. The table exposes fallback need as a need, never as a fetched fact.

## Maintenance

- When a slice resolves an entry, set Status to `Resolved (<slice name>, <date>)` and keep the row.
- When a new deferral is made in a handoff addendum, add a row here in the same slice so the two stay in sync.
- This ledger does not replace the handoff log; it is the durable index of open deferrals the handoff entries create.

## References

- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md` (station boundaries, memory section)
- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md` (Probe: requests, does not retrieve)
- `02 Platform/architecture/cloud-tool-runtime-plan.md` (Tool Gateway, Functions, pgvector deferrals)
- `06 Execution/handoffs/current-slice.md` (the probe-refresh chain addenda this ledger indexes)
