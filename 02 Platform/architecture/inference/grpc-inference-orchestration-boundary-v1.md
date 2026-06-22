# gRPC Inference Orchestration Boundary v1

**date:** 2026-06-22
**status:** DIRECTIONAL ARCHITECTURE PLAN. Vault-only. No application code, prompt, runtime, proto,
migration, buyer-UI, or reconciliation change. gRPC is NOT introduced by this slice.
**relation to prior doctrine:** extends `02 Platform/architecture/next-platform-architecture-plan.md` §5
("where http stays and where grpc makes sense later") with the multi-stage-inference angle. Does not
override it. Reconciles with the existing FastAPI gRPC scaffold on port 50051 (`protos/assist.proto`),
which `cloud-deploy-readiness-v1.md` and `azure-container-apps-provisioning-plan-v1.md` record as present
but unexposed and unused on the sports path.

## 1. Purpose

Define, ahead of need, where gRPC would and would not create leverage inside DAI once inference becomes
multi-stage (primary analysis -> verifier/critic -> synthesis), so that today's in-process C# DTOs and
source envelopes are shaped to be *promotable* to protobuf later -- without building gRPC now and without
distracting from the source-depth track. This is a contract-shaping and boundary-selection plan, not an
implementation slice.

The problem it solves: when the inference path splits into separate model passes across a process
boundary, an ad-hoc REST/JSON contract between the orchestrator and the inference worker would drift,
lose typing, and make verifier/synthesis findings hard to version. Deciding the boundary and the
conceptual contracts now prevents that drift later.

## 2. Current Context

- **Source-depth expansion is the immediate pipeline-improvement track.** Market Odds Depth v1 (just
  shipped) deepened the market signal; MLB source-depth work continues. That work raises the quality of
  evidence *entering* the factory and is independent of transport choice.
- **Reconciliation/calibration loop exists and works** (outcome reconciliation runtime + calibration
  reads). It is HTTP/manual today and should stay that way.
- **Inference is single-stage today.** `AgentRunService.RunSportsMatchupPipelineAsync` runs four explicit
  steps -- retrieve, analyze, evaluate, compose (`orchestration.md`). The only model call is one
  `POST /api/sports/analyze` from `FastApiClient` to the FastAPI agent-service over HTTP, routed through
  the live `ToolGateway`. Evaluate and compose are deterministic, in-process C#.
- **Multi-model inference is emerging as a future need** (verifier/critic pass, synthesis pass, grounding/
  conflict checks). It is not built. `ProtocolNodeRunner` and `ProbeRefreshExecutor` remain dormant; the
  Cognitive Protocol is model-emitted + platform-completed, not a multi-process runtime.
- **gRPC here is directional planning only.** A gRPC listener is already scaffolded in FastAPI (50051,
  `assist.proto`) but unexposed and unused. This plan does not turn it on.

## 3. Where gRPC Has Leverage

The leverage condition (from `next-platform-architecture-plan.md` §5) is: an internal service boundary
where typed contracts, versioning, smaller payloads, or streaming add real value. Multi-stage inference
is the first credible trigger. Candidates, in priority order:

- **.NET orchestrator <-> inference service (strongest candidate).** Once analysis is more than one model
  call, a typed, versioned contract between `DevCore.Api` and a dedicated inference worker beats an
  expanding REST/JSON surface. This is the boundary to design first.
- **Primary analysis request/response.** A `DecisionAnalysisRequest` carrying source envelopes + run
  identity, returning a proposed artifact. Typed, versioned, high call volume -> good protobuf fit.
- **Verifier/critic findings.** A structured `ArtifactVerificationResponse` (grounding, conflict,
  overclaiming findings) is exactly the kind of rigid, enumerated contract gRPC serves well.
- **Synthesis response.** Integrating source envelopes + primary output + deterministic checks + verifier
  findings into the final artifact benefits from a stable typed contract.
- **Future agent worker contracts.** If reusable worker roles (collector/evaluator/synthesizer) ever run
  out-of-process, their boundaries are gRPC candidates -- but only once the role contracts stabilize.
- **Optional internal run-progress events (later).** Server-streaming from inference worker to orchestrator
  for stage progress, instead of polling -- the second trigger condition in §5. Defer until there is a UX
  or operational reason to stream.

## 4. Where gRPC Does Not Have Leverage Yet

- **External source ingestion.** odds_api, mlb_statsapi, espn, actionnetwork are provider-facing REST/JSON.
  These stay REST adapter clients; gRPC has zero leverage here (we do not own the provider contracts).
- **Angular / buyer frontend.** REST stays the right Angular -> .NET boundary (§5, `frontend-philosophy.md`).
  No gRPC-Web.
- **Reconciliation endpoints.** Manual/HTTP `POST /api/agent-runs/reconcile` etc. are low-volume, human-in-
  the-loop, and stable as HTTP.
- **Docs/calibration workflow.** Scripts + vault docs; no service boundary.
- **Unstable/experimental protocol nodes.** `ProtocolNodeRunner`, `ProbeRefreshExecutor`, and the
  discern/probe internals are dormant and still-evolving; formalizing them as wire contracts now would
  lock in an unstable workflow. Keep them in-process until they earn a stable shape.

## 5. Proposed Boundary

The first recommended (future) boundary is the orchestrator -> inference worker split, keeping persistence
and delivery in .NET:

```
  buyer (Angular)            external providers
        | HTTP                    | REST/JSON (provider adapters)
        v                         v
  +-------------------------------------------+
  |  DevCore.Api  (orchestrator / platform)   |
  |  - run dispatch, retrieve (source clients)|
  |  - deterministic evaluate + compose       |
  |  - persistence, reconciliation, delivery  |
  +-------------------------------------------+
        |  gRPC (typed, versioned)  ^  findings/response
        v   internal boundary       |
  +-------------------------------------------+
  |  Inference Worker (model passes)          |
  |   1. primary model  -> proposed artifact  |
  |   2. verifier model -> findings           |
  |   3. synthesis pass -> final composition  |
  +-------------------------------------------+
```

Notes on the boundary:
- Retrieval/source acquisition stays in `DevCore.Api` (or its REST adapters), not behind gRPC -- gRPC
  carries *already-acquired* source envelopes into inference, it does not fetch sources.
- Deterministic evaluate/compose, persistence, reconciliation, and delivery stay in .NET.
- The worker is stateless per request; tenant/run identity travels in the request metadata.
- This is one boundary, not an agent-runtime rewrite.

## 6. Conceptual Contract Shapes

Conceptual messages only -- NOT `.proto` files. Today these are in-process C# records (some already exist:
`SportsRetrievalOutput`, `SourceDepthRecord`, `SignalAvailabilityRecord`, `AgentRunExecutionResult`).
Shape them so promotion to protobuf is mechanical.

- **DecisionAnalysisRequest** -- runId, tenantId, correlationId, runType/niche, competition, matchup
  identity (sourceProvider, externalGameId), `SourceSignalEnvelope[]`, requested artifact version, model
  hint/policy.
- **DecisionAnalysisResponse** -- proposed artifact (lean, leanSide, summary, factors, counterCase,
  watchFor, whatWouldChange), analyzer confidence, model id, protocol seed, evidenceRichness, latency/cost.
- **ArtifactVerificationRequest** -- runId, the proposed artifact, the `SourceSignalEnvelope[]` it was
  built from, deterministic check outputs (direction consistency, named-risk grounding, source depth).
- **ArtifactVerificationResponse** -- `VerificationFinding[]`, overall verdict (pass / caution / block-
  publish), model id, latency/cost. Findings, not a re-pick.
- **SynthesisRequest** -- runId, primary artifact, verifier findings, source envelopes, deterministic
  checks.
- **SynthesisResponse** -- final composed artifact fields, `SynthesisSummary`/`SynthesisTrace`
  (what was integrated, what was down-weighted, what was left absent), model id, cost.
- **SourceSignalEnvelope** -- signal name, source group, provider, status (grounded/missing/proxy/
  unavailable), depth level (none/identity_only/shallow/enriched), observed flag, fetchedAt/updatedAt,
  provenance, payload (typed per signal). This is the promotion target of today's retrieval output +
  `SourceDepthRecord` + `SignalAvailabilityRecord`.
- **VerificationFinding** -- finding type (ungrounded_named_risk, direction_conflict, overclaim,
  source_insufficient), section, severity, referenced signal/group, human-readable reason. (Maps onto
  today's NamedRiskGrounding / ArtifactDirectionConsistency observations.)
- **SynthesisTrace / SynthesisSummary** -- integratedInputs, downWeighted, unresolvedGaps, final lean
  rationale; observed-only, auditable.

## 7. Model-to-Model Orchestrated Inference

Intended pattern, with guardrails:

- **Primary model proposes** the artifact (lean, factors, counter-case) from the source envelopes.
- **Verifier checks** grounding, conflicts, and overclaiming against the *same* envelopes; it emits
  findings only.
- **Synthesis integrates** source envelopes + primary output + deterministic checks + verifier findings
  into the final artifact.

Hard rules (preserve the two-axis doctrine and auditability):
- The verifier does NOT make an independent pick. It cannot set or flip `LeanSide`; it can only flag.
- Synthesis does NOT invent missing evidence. Absent signals stay absent; it may down-weight or add
  humility, never fabricate.
- **Structured `LeanSide` stays controlled and auditable** -- proposed by primary, never silently changed
  by verifier/synthesis; any change is an explicit, logged synthesis decision with a trace.
- Confidence remains coherence, calibrated deterministically in .NET (the evaluator), not by the verifier.
- Source depth / evidence sufficiency remain separate from confidence.

## 8. Versioning and Compatibility

- **Artifact contract versioning continues** (`sports_decision_artifact_v3` today). The wire contract
  version is independent of the artifact version; both travel in messages.
- **Protobuf strategy (when adopted):** additive field numbers only; never renumber or reuse; reserve
  removed fields; new optional fields default-safe. Generated stubs: .NET via `Grpc.Tools`, Python via
  `grpcio-tools` (per §5).
- **Backward compatibility:** the orchestrator must accept responses missing newer optional fields and
  treat them as absent, not as errors.
- **Fail-soft:** if the inference worker is unreachable or a stage fails, the run degrades to the best
  available stage (e.g. primary-only, no verifier) and records the degradation -- it never fabricates a
  verifier/synthesis result. Mirrors today's fail-soft source behavior.
- **DTOs now, protobuf later:** keep request/response/envelope shapes as clean C# records with no HTTP- or
  JSON-specific assumptions so promotion is mechanical.

## 9. Observability and Safety

- Propagate **requestId, runId (X-Agent-Run-Id is the canonical anchor), tenantId, modelId** through every
  message and log line.
- Carry **source provenance** (provider, fetchedAt) and **verifier findings** in the persisted artifact /
  trace.
- **Latency and cost** logged per stage (primary / verifier / synthesis), per model id.
- **No cross-tenant access:** tenant identity is required on every inference request; the worker is
  stateless and never reads another tenant's data. (Aligns with `security-and-permissions.md`.)
- **No implicit tool permissions:** the inference worker receives only the envelopes handed to it; it does
  not acquire sources or invoke tools. Tool authorization stays with the Tool Gateway in .NET.
- Billing hooks: per-stage model cost is already a recorded dimension; keep it on the response so a future
  metering/Stripe path can read it without re-architecting.

## 10. Deferred Decisions

Explicitly deferred (NOT now):
- implementing gRPC at all;
- migrating the current FastAPI `POST /api/sports/analyze` boundary;
- replacing any working HTTP API;
- gRPC-Web / any browser-facing gRPC;
- a full agent-worker runtime;
- streaming run-progress events;
- Protocol Node Runner activation;
- Probe Refresh activation;
- exposing the existing FastAPI 50051 listener.

## 11. Recommended Follow-Up

**Keep building the MLB source-depth track now** (it improves the evidence entering the factory and is
transport-agnostic). gRPC remains unbuilt. The first inference-contract work should wait until source
envelopes and verifier needs stabilize; when they do, run a narrow **Inference Worker Contract v1** -- C#
record contracts for the orchestrator <-> inference boundary (primary/verifier/synthesis), still in-process
or over the existing HTTP, explicitly shaped for later protobuf promotion. Introduce gRPC transport only
when the §5 trigger is real (multi-stage split across processes, or a streaming need), not anticipated.

## 12. Ready-to-Paste Future Implementation Prompt

> FUTURE / NOT NOW. Do not run until source envelopes + a verifier pass are real and the §5 gRPC trigger
> condition is met (multi-stage inference across a process boundary, or a streaming need).

```
[FUTURE SLICE -- DO NOT EXECUTE UNTIL TRIGGERED]
Slice: Inference Worker Contract v1

Goal: define typed C# record contracts for the DevCore.Api orchestrator <-> inference worker boundary
(primary analysis, verifier findings, synthesis), shaped for later protobuf promotion. Transport stays
in-process or over the existing HTTP for now; no gRPC yet.

Skills gate: dai-skill-router, dai-grill-with-vault, superpowers:test-driven-development,
dai-test-discipline, verification-before-completion, dai-agent-handoff.

Repos: dai (code), dai-vault (docs).

Scope:
- C# records: DecisionAnalysisRequest/Response, ArtifactVerificationRequest/Response,
  SynthesisRequest/Response, SourceSignalEnvelope, VerificationFinding, SynthesisTrace.
- Promote today's retrieval output + SourceDepthRecord + SignalAvailabilityRecord into SourceSignalEnvelope.
- Verifier emits findings only; never sets/flips LeanSide. Synthesis never fabricates absent evidence.
- Fail-soft: missing verifier/synthesis stage degrades to best available, recorded, never invented.

Non-goals: no gRPC transport, no .proto, no FastAPI boundary migration, no buyer UI, no migration,
no confidence/posture/lean logic change, no Protocol Node Runner / Probe Refresh activation.

Tests (TDD): contract shape + fail-soft + LeanSide-immutability-by-verifier. Verify dotnet test green.
Trigger check first: confirm the §5 gRPC condition (multi-stage split or streaming need) is actually real
before adding any transport. If not real, stop -- contracts only.
```
