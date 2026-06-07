# Factory Line Balance v1

**date:** 2026-06-06
**status:** architecture note. vault-first review. documentation only.
**scope:** current DAI decision factory line, station ownership, station boundaries, and next hardening priority after Discern Station Runner Groundwork v1.

## Executive Finding

The current bottleneck is not another dormant runner. The line now has enough runner-shaped groundwork to prove the pattern: `interrogate.probe` has deterministic execution, Perceive has normalized observation contracts and a collector, and Discern has a dormant station runner shape. None of those should be activated by default.

The next factory hardening priority is **Protocol Station Contract Completion v1**: finish the machine-readable station ownership contract before any more production adoption. The prose doctrine is strong, but `ProtocolRegistry v0` still encodes only the subset that was honest at the time: id, macro, micro-action, purpose, allowed tools, model-call policy, cost class, telemetry tags, quality gates, and calibration hooks. It does not yet encode artifact fields read/written, allowed scripts/reflexes, input/output contracts, fallback behavior, forbidden behavior, token budget, uncertainty/skip statuses, or artifact mutation policy.

That gap matters more than activating Discern. If a future slice wires `DiscernStationRunner`, Perceive intake, memory tools, document tools, or probe-refresh adoption before the station cards carry full ownership boundaries, the platform will be executing from partial contracts.

## Repository Posture Before Changes

- <DAI_REPO_ROOT>: `main...origin/main [ahead 3]`, clean.
- <DAI_VAULT_ROOT>: `main...origin/main [ahead 3]`, clean.
- <JERA_SKILLS_ROOT>: `main...origin/main`, clean.

## Skills And Guidance Used

- Local <JERA_SKILLS_ROOT>/dai, read-only: dai-grill-with-vault, dai-agent-handoff, dai-token-tight, dai-write-skill boundary guidance.
- Built-in guidance applied manually: planning, verification-before-completion.
- test-driven-development was not used because this slice changed documentation only.
- systematic-debugging was not needed; no runtime defect surfaced.

## Docs Reviewed

- `02 Platform/decisions/0004-cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/cognitive-protocol-runtime.md`
- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md`
- `02 Platform/architecture/cognitive-factory/factory-line-balance-review-v1.md`
- `02 Platform/architecture/cognitive-factory/fastapi-canonical-field-migration-plan-v1.md`
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md`
- `02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md`
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `02 Platform/architecture/cloud-tool-runtime-plan.md`
- `04 Products/sports-v1/decision-factory-hardening-v1.md`
- `04 Products/sports-v1/calibration/20260528-0931-nba-canonical-protocol-spot-check.md`
- `04 Products/sports-v1/calibration/protocol-runs/2026-05-18-cognitive-protocol-outcome-reconciliation.md`
- `06 Execution/handoffs/current-slice.md`

Code was spot-checked only to prevent stale-doc carryover: `CognitiveProtocol`, `CognitiveProtocolBuilder`, `ProtocolRegistry`, `ProtocolNodeRunner`, `ProtocolDiagnosticsRollup`, `PerceiveSignalObservationCollector`, `DiscernStationRunner`, `SportsQualityChecker`, and `SignalFollowUpEvaluator`.

## Current Factory Model

- Platform = factory. It owns orchestration, station roles, permissions, diagnostics, artifact shape, usage, tenancy, billing hooks, and repeatable safety policy.
- Agents = workers. They execute bounded jobs inside platform-owned contracts.
- Tenants = businesses/workspaces. Tenant identity governs enablement, cost, audit, and entitlement.
- Niches = assembly lines. They provide source strategy, signal taxonomy, prompt content, posture vocabulary, scoring thresholds, and product-specific copy.
- Frontends = packaging. They render the artifact and must not redefine station semantics.
- Stripe = truth. Economic gates come from billing truth, not ad hoc feature flags.

The current runtime writes canonical `CognitiveProtocol` records. The model emits an analyzer protocol seed with 11 cognitive station outputs in one analyze call. The platform completes the protocol by adding deterministic `interrogate.probe` and the Synthesize trio.

## Factory Line Summary

| Segment | Current implementation | Ownership state | Activation state |
|---|---|---|---|
| Perceive | model-emitted Detect/Frame/Aim inside shared analyze; normalized signal contracts exist dormant | prose mature; runtime contracts partial | no generic production intake consumer |
| Interrogate.Question | model-emitted counter-case | prose mature; prompt/quality guarded | active in shared analyze |
| Interrogate.Probe | deterministic platform completion from `SignalFollowUpRecord[]`; structured `ProbeRequest` exists | most mature station; refresh chain overdeep but dormant | probe output active; refresh loop inactive |
| Interrogate.Verify | model-emitted alternate explanation | prose mature; prompt/quality guarded | active in shared analyze |
| Discern.Weigh | deterministic signal grades plus model narrative | deterministic backbone mature; runner dormant | active narrative; runner inactive |
| Discern.Contrast | model-emitted signal alignment/divergence | partial; thin when only one signal grounds | active in shared analyze |
| Discern.Stress | model-emitted fragility; canonical single source | mature vocabulary; runner dormant | active in shared analyze; runner inactive |
| Decide.Resolve | model-emitted direction text plus platform lean-side clamp | partial; clamp mature | active in shared analyze |
| Decide.Position | model-emitted posture proposal plus platform enum clamp | strong boundary; generic policy guard deferred | active in shared analyze |
| Decide.Justify | model-emitted rationale; deterministic confidence number/band | confidence boundary mature; calibration feedback offline | active in shared analyze |
| Synthesize.Integrate | deterministic platform operation | mature | active |
| Synthesize.Compose | deterministic artifact assembly | mature | active |
| Synthesize.Deliver | deterministic DTO/persistence boundary | mature for initial artifact; mutation writer deferred | active |

## Maturity Scale

- **Mature:** production behavior exists, ownership is clear, and regression tests or deterministic guards pin it.
- **Partial:** doctrine and some runtime surface exist, but production behavior is still shared-analyze, dormant, or missing a full station-card contract.
- **Thin:** prose exists but runtime or inspection contracts are underbuilt.
- **Overdeep:** dormant scaffolding is deeper than current product/runtime need.

## Station Operating Map

### Perceive

This note groups `perceive.detect`, `perceive.frame`, and `perceive.aim` because the current factory bottleneck is Perceive intake ownership as a line segment. The runtime still has three station cards.

1. **Purpose:** surface what is grounded, what is missing, and what matters.
2. **Owns:** interpretation of already-stamped retrieval output; normalized signal observation language for future intake.
3. **Must not own:** retrieval, tool selection, provider calls, posture, confidence, or domain-specific source policy.
4. **Inputs:** user/run input, grounded signals, missing signals, signal availability, retrieval context blocks, optional future normalized observations.
5. **Outputs:** `perceive.detect`, `perceive.frame`, `perceive.aim`; dormant `PerceiveSignalObservation` projections/sets.
6. **Artifact mutation:** no direct persisted artifact mutation. It produces bounded station output that Synthesize composes.
7. **Tool calls:** no direct tools. Retrieval is platform plumbing before Perceive. The model-emitted fields are served by the shared analyze call today.
8. **Model calls:** yes, inside the shared analyze call today.
9. **Deterministic checks/reflexes:** signal-used integrity, grounded-factor checks, frame context calibration flags, Perceive signal intake validation when used manually.
10. **Uncertainty/failure/skip states:** no grounded signals, missing signal context, empty/invalid normalized observation, unsupported/no-signal intake.
11. **Current maturity:** partial. Prose and contracts are strong; production adoption of generic intake is still deferred.
12. **Next risk/opportunity:** complete station-card read/write/input/fallback fields before routing more inputs through Perceive. Do not let probe-refresh sports payload logic become the platform intake layer.

### Interrogate.Question

1. **Purpose:** name the strongest open question or counter-case against the emerging read.
2. **Owns:** pressure-testing the initial read using grounded or missing signal evidence.
3. **Must not own:** retrieval, fallback equivalence grading, final stance, confidence, posture, or new facts.
4. **Inputs:** Perceive outputs, grounded/missing signals, signal availability.
5. **Outputs:** `interrogate.question`.
6. **Artifact mutation:** no direct persisted artifact mutation; produces bounded station output.
7. **Tool calls:** no direct tools; shared analyze is the producing stage.
8. **Model calls:** yes, inside shared analyze.
9. **Deterministic checks/reflexes:** guardrails for generic counter-case language and unsupported claims; calibration flag `counter_case_generic`.
10. **Uncertainty/failure/skip states:** thin evidence should emit the limitation as the counter-case; blank/generic counter-case is a quality risk.
11. **Current maturity:** partial to mature. It is active and guarded, but still model-emitted inside a shared call.
12. **Next risk/opportunity:** keep Question from absorbing Probe's "what to investigate" or Discern's "how much it is worth" responsibilities.

### Interrogate.Probe

1. **Purpose:** summarize missing primary signals worth investigating.
2. **Owns:** deterministic gap summary and structured probe request from signal follow-ups.
3. **Must not own:** retrieval execution, Perceive refresh, Tool Gateway calls, memory lookup by default, artifact mutation, or confidence/posture changes.
4. **Inputs:** `SignalFollowUpRecord[]`.
5. **Outputs:** `interrogate.probe`; structured `ProbeRequest`.
6. **Artifact mutation:** no direct mutation. Probe output is composed into the current artifact; refresh mutation remains deferred.
7. **Tool calls:** none today. Future memory/retrieve tools remain gated and deferred.
8. **Model calls:** no.
9. **Deterministic checks/reflexes:** missing-primary detection, template-known signal filter, no `line_movement` probe, deterministic ordering.
10. **Uncertainty/failure/skip states:** no follow-ups, no missing primary, or no doctrinal template -> null/no probe material.
11. **Current maturity:** mature for deterministic output; overdeep around dormant refresh chain.
12. **Next risk/opportunity:** stop deepening probe-refresh until station contracts are complete. Any future memory-backed Probe must be a Tool Gateway-governed, tenant-gated slice.

### Interrogate.Verify

1. **Purpose:** offer an alternate explanation tested against staged evidence.
2. **Owns:** grounded alternate explanations.
3. **Must not own:** new evidence claims, retrieval, final stance, confidence, posture, or stress/fragility.
4. **Inputs:** Perceive outputs, Question, signal availability.
5. **Outputs:** `interrogate.verify`.
6. **Artifact mutation:** no direct mutation; bounded station output only.
7. **Tool calls:** no direct tools.
8. **Model calls:** yes, inside shared analyze.
9. **Deterministic checks/reflexes:** reframe guardrail against hidden strengths, form, travel, weather, injuries, or availability without grounding.
10. **Uncertainty/failure/skip states:** no grounded alternate -> state that plainly.
11. **Current maturity:** partial.
12. **Next risk/opportunity:** keep Verify from becoming ungrounded narrative filler; expose its fallback behavior in full station cards.

### Discern.Weigh

1. **Purpose:** grade signals by quality and decision-use; separate grounded evidence from weak or absent evidence.
2. **Owns:** evidence quality hierarchy and deterministic grading interpretation.
3. **Must not own:** final posture, confidence number, retrieval, or domain payload fetching.
4. **Inputs:** signal availability, grounded/missing signals, fallback ladder fields, optional normalized Perceive observations for dormant runner inspection.
5. **Outputs:** `discern.weigh`; deterministic grades already live on signal availability records.
6. **Artifact mutation:** no direct persisted mutation.
7. **Tool calls:** no direct tools. `SignalQualityEvaluator` is a deterministic platform reflex, not a tool call.
8. **Model calls:** yes for narrative only, inside shared analyze. The dormant `DiscernStationRunner` makes no model call.
9. **Deterministic checks/reflexes:** `SignalQualityEvaluator`, fallback ladder classification, anti-hype signal effect, Discern narrative guardrail.
10. **Uncertainty/failure/skip states:** no signals -> say nothing is grounded; dormant runner can emit `NoInput`, `UnsupportedStation`, or `InvalidInput`.
11. **Current maturity:** partial. Deterministic backbone is mature; platform runner is dormant and shallow by design.
12. **Next risk/opportunity:** do not activate the runner without a caller. Complete station-card contracts so Weigh's deterministic/script ownership is explicit.

### Discern.Contrast

1. **Purpose:** interpret alignment or divergence between grounded signals.
2. **Owns:** cross-signal relationship interpretation.
3. **Must not own:** single-signal overclaiming, final posture, confidence number, or fallback equivalence rules.
4. **Inputs:** grounded signals, signal availability, Perceive outputs.
5. **Outputs:** `discern.contrast`.
6. **Artifact mutation:** no direct mutation.
7. **Tool calls:** no direct tools.
8. **Model calls:** yes, inside shared analyze.
9. **Deterministic checks/reflexes:** contrast-missing calibration flag when external signals are supplied.
10. **Uncertainty/failure/skip states:** one grounded signal -> limited or no contrast; null contrast with external blocks is a quality risk.
11. **Current maturity:** partial/thin. It is active, but its meaningfulness depends heavily on signal richness.
12. **Next risk/opportunity:** encode the single-signal fallback rule and contrast prerequisites in station-card input/fallback contracts.

### Discern.Stress

1. **Purpose:** name the fragility, key risk, or condition under which the read fails.
2. **Owns:** fragility framing after evidence is weighed.
3. **Must not own:** counter-case questioning, retrieval, final stance, confidence/posture mutation, or fabricated injury/form risk.
4. **Inputs:** Perceive outputs, Discern.Weigh, Discern.Contrast, missing signals.
5. **Outputs:** `discern.stress`.
6. **Artifact mutation:** no direct mutation.
7. **Tool calls:** no direct tools.
8. **Model calls:** yes, inside shared analyze.
9. **Deterministic checks/reflexes:** canonical single-source stress, no legacy concatenation, stress guardrail for concrete fragility.
10. **Uncertainty/failure/skip states:** at minimum, "thin evidence base" is valid; vague fragility is a quality risk.
11. **Current maturity:** partial/mature. Vocabulary and canonical location are stable; runner remains dormant.
12. **Next risk/opportunity:** keep Stress in Discern and avoid pulling it back into Interrogate or Synthesize copy.

### Decide.Resolve

1. **Purpose:** reconcile weighed evidence into a single direction or non-direction without hype.
2. **Owns:** direction text and evidence-bound resolution.
3. **Must not own:** posture enum authority, confidence math, tool calls, or new evidence.
4. **Inputs:** Discern outputs and Perceive outputs.
5. **Outputs:** `decide.resolve`; contributes to lean/lean-side extraction.
6. **Artifact mutation:** no direct mutation.
7. **Tool calls:** no direct tools.
8. **Model calls:** yes, inside shared analyze.
9. **Deterministic checks/reflexes:** lean-side validation/clamp to valid direction or null.
10. **Uncertainty/failure/skip states:** split signals -> non-direction and null lean side.
11. **Current maturity:** partial. Active, but still shared-analyze and model-text driven.
12. **Next risk/opportunity:** ensure Resolve does not become a pick. Keep product/frontends on "read stance" language.

### Decide.Position

1. **Purpose:** set the decision posture from the niche posture vocabulary.
2. **Owns:** bounded posture value after platform validation.
3. **Must not own:** pick/lock framing, confidence math, unbounded posture values, or override of signal blocks.
4. **Inputs:** Discern outputs, Resolve, confidence, signal availability, confidence effects.
5. **Outputs:** `decide.position`.
6. **Artifact mutation:** no direct mutation.
7. **Tool calls:** no direct tools.
8. **Model calls:** yes, inside shared analyze; platform is final enum authority.
9. **Deterministic checks/reflexes:** posture enum clamp; anti-hype warning when `play` conflicts with signal-layer aggressive block.
10. **Uncertainty/failure/skip states:** invalid/absent posture -> null/warning; aggressive posture with partial evidence -> quality/calibration flag.
11. **Current maturity:** mature boundary, partial generic policy. Probe-refresh merge guards protected fields, but a generic Decide policy guard is deferred until a write path appears.
12. **Next risk/opportunity:** encode mutation policy in station cards; do not build confidence/posture mutation until calibration and write-governance deferrals resolve.

### Decide.Justify

1. **Purpose:** explain why deterministic confidence fits the evidence.
2. **Owns:** calibration rationale sentence.
3. **Must not own:** confidence number, confidence band, threshold changes, or posture mutation.
4. **Inputs:** deterministic confidence, confidence band, evidence richness, analyzer confidence, Discern.Weigh.
5. **Outputs:** `decide.justify`.
6. **Artifact mutation:** no direct mutation.
7. **Tool calls:** no direct tools.
8. **Model calls:** yes, inside shared analyze for the sentence only.
9. **Deterministic checks/reflexes:** `SportsEvaluator` confidence/band; offline calibration flag `confidence_high_for_partial_evidence`.
10. **Uncertainty/failure/skip states:** zero/thin grounded evidence -> dampened confidence and prior-only rationale.
11. **Current maturity:** partial/mature. Confidence ownership is clear; runtime warning for weak-evidence confidence remains deferred.
12. **Next risk/opportunity:** do not auto-feed calibration into confidence. First make station calibration hooks and quality-gate surfaces complete and inspectable.

### Synthesize.Integrate

1. **Purpose:** combine validated prior-protocol material without adding new claims.
2. **Owns:** deterministic integration of already-produced material.
3. **Must not own:** model reasoning, new facts, retrieval, confidence mutation, or posture mutation.
4. **Inputs:** analyzer output, retrieval output, evaluator result, quality warnings, completed protocol.
5. **Outputs:** `synthesize.integrate`; integrated execution result material.
6. **Artifact mutation:** no post-hoc mutation; participates in initial artifact assembly.
7. **Tool calls:** none.
8. **Model calls:** no.
9. **Deterministic checks/reflexes:** carry quality warnings; no-new-claims discipline.
10. **Uncertainty/failure/skip states:** analyze failure routes to failed artifact path.
11. **Current maturity:** mature.
12. **Next risk/opportunity:** preserve Synthesize as platform-operational; do not simulate cognition here.

### Synthesize.Compose

1. **Purpose:** assemble the final decision artifact shape.
2. **Owns:** artifact shape, version stamping, completed `CognitiveProtocol`, deterministic Probe completion, Synthesize text constants.
3. **Must not own:** model calls, new facts, provider calls, or hidden schema drift.
4. **Inputs:** integrated material and analyzer protocol seed.
5. **Outputs:** `synthesize.compose`; `AgentRunExecutionResult`; canonical `CognitiveProtocol`; artifact version.
6. **Artifact mutation:** creates/composes the run artifact; does not mutate an existing persisted artifact.
7. **Tool calls:** none.
8. **Model calls:** no.
9. **Deterministic checks/reflexes:** `CognitiveProtocolBuilder.FromAnalyzerProtocolSeed`, `BuildProbe`, artifact shape consistency.
10. **Uncertainty/failure/skip states:** failure path stamps artifact era and leaves `CognitiveProtocol` null.
11. **Current maturity:** mature.
12. **Next risk/opportunity:** station-card completion should encode Compose's write scope so future merge writers cannot blur initial composition with post-hoc mutation.

### Synthesize.Deliver

1. **Purpose:** return the consumable result and persist the full artifact.
2. **Owns:** delivery DTO mapping, persisted `OutputJson`, compact customer-facing result boundary.
3. **Must not own:** station reasoning, new claims, raw prompts/provider metadata, or product copy that relabels posture as a pick.
4. **Inputs:** composed artifact.
5. **Outputs:** user-facing result, persisted artifact, inspection-only internal fields.
6. **Artifact mutation:** persists the newly composed artifact. It must not be used as a refresh/merge writer.
7. **Tool calls:** none.
8. **Model calls:** no.
9. **Deterministic checks/reflexes:** internal fields excluded from customer DTO; artifact inspection keeps quality fields visible; failure artifact persistence.
10. **Uncertainty/failure/skip states:** failed run persists failure artifact and pipeline metadata.
11. **Current maturity:** mature for initial delivery; merge writer remains deferred.
12. **Next risk/opportunity:** maintain the separation between initial persistence and any future audited artifact mutation path.

## Boundary Challenges

### Are We Overbuilding Dormant Runners?

Yes, if the next slice activates or expands runner work. `ProtocolNodeRunner` supports only `interrogate.probe`; `DiscernStationRunner` is intentionally dormant; Perceive observation collection is read-only. The line does not need another runner before the station ownership contract is complete.

### Are Station Ownership Boundaries Clear Enough?

In prose, yes. In runtime metadata, not yet. `protocol-node-specs.md` and this note define ownership clearly, but `ProtocolStationCard` does not yet encode read/write fields, input/output contracts, scripts, fallback behavior, forbidden behavior, token budgets, or station uncertainty statuses.

### Is Artifact Mutation Governance Clear?

For probe-refresh, yes: planner, review, dry-run, audit, store, and diagnostics all protect the boundary, and no writer exists. For the whole station line, the missing piece is explicit `may_mutate_artifact` or `artifact_write_mode` on the station contract so future writers cannot confuse Synthesize.Deliver with post-hoc mutation.

### Are Confidence And Posture Mutation Rules Clear?

Yes. Confidence is deterministic; posture is a clamped enum; confidence/posture/lean mutation remains deferred and calibration-gated. The generic guard should wait until a real non-probe write path or merge writer appears.

### Are Fallback/Proxy Signal Rules Clear?

Mostly. The signal fallback ladder clearly separates Interrogate (candidate), Discern (equivalence/worth), and Decide (permission). The remaining product artifact gap is explicit proxy-label surfacing, already named in sports hardening docs. It is a sports product hardening slice, not the factory bottleneck.

### Is Calibration Feedback Connected Enough?

It is connected for offline review, not automatic runtime change. That is correct for now. Calibration flags should continue to inform quality gates and station-card calibration hooks before any confidence threshold changes.

### Is Any Station Doing Work That Belongs Elsewhere?

No major current violation found. The main risk is future erosion:
- Perceive must not retrieve.
- Probe must not call tools directly.
- Discern must not set confidence/posture.
- Decide must not mutate confidence/posture without calibration proof.
- Synthesize must not invent claims or act as a merge writer.

### Missing Documentation Or Contract Before Runtime Adoption?

Yes: the machine-readable station contract is incomplete. The durable prose map exists now, but runtime adoption should wait until the station card contract carries the boundaries the prose already knows.

## Deferred Decisions Reaffirmed

- Direct Interrogate -> Perceive self-invocation remains deferred/forbidden.
- Probe-refresh executor activation remains deferred.
- Probe-refresh artifact mutation and merge writer remain deferred.
- Confidence/posture/lean mutation remains deferred and calibration-gated.
- Decide recommendation application remains deferred.
- Synthesize preview surfacing remains deferred.
- Memory/pgvector and document tools remain deferred and gateway-governed.
- Discern station runner adoption remains deferred; no concrete read-only caller was found in this review.

## Next Recommended Slice

**Protocol Station Contract Completion v1.**

Complete the runtime station-card contract without activating stations. Add the missing ownership fields from the blueprint and this note to the station manifest and diagnostics:

- artifact fields read
- artifact fields written
- input contract
- output contract
- allowed scripts/reflexes
- fallback behavior
- forbidden behavior
- token budget
- allowed memory queries
- station uncertainty/skip/failure statuses
- artifact mutation policy

Why this is the current bottleneck:

- It strengthens the whole factory line instead of deepening the newest dormant runner.
- It makes future Perceive, Discern, Decide, memory, and document-tool adoption safer.
- It gives diagnostics and startup validation a complete ownership surface.
- It keeps behavior unchanged: no endpoint, no model call, no Tool Gateway behavior change, no artifact mutation, no schema change, no FastAPI/Angular change.

Backup if product artifact quality is chosen over factory hardening: **Proxy Label Surfacing v1** from `decision-factory-hardening-v1.md`. That is a valid sports product slice, but it is not the platform factory bottleneck.

### Progress Update: Protocol Station Contract Completion v1

Implemented 2026-06-06 as a metadata-only runtime contract slice. `ProtocolStationCard` now carries the v1 station ownership surface: execution owner, runtime maturity, artifact mutation policy, artifact fields read/written, input/output contracts, allowed scripts/reflexes, allowed memory queries, fallback behavior, forbidden behavior, and token budget. All 15 default station cards are populated, and validator/diagnostics/rollup surfaces expose the new fields.

This does not activate stations, split model calls, change Tool Gateway behavior, mutate artifacts, add endpoints, change schema, change FastAPI prompts, change Angular, or add a merge writer. Station uncertainty/skip/failure statuses remain the next doc-to-runtime metadata gap before any broader runtime adoption.

## Anti-Goals For The Next Slice

- Do not activate `DiscernStationRunner`.
- Do not route production Perceive intake through the collector.
- Do not wire probe-refresh into production.
- Do not build a merge writer.
- Do not add endpoints.
- Do not change FastAPI prompts or model call count.
- Do not change Angular.
- Do not change Tool Gateway behavior.
- Do not mutate artifacts.
- Do not change confidence/posture/lean rules.
- Do not add schema migrations.
- Do not hard-code sports-specific source rules into core platform services.

## Verification For This Note

- Documentation-only change.
- No runtime files changed.
- Deferred ledger updated only for the newly identified station-card contract completion decision.
- Exact local path scan required before completion.
- Markdown diff review required before completion.
