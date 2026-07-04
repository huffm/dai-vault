---
title: "Cognitive Protocol Mapping Closeout v1"
type: "evidence-report"
date: "2026-07-04"
status: "active"
project: "DAI"
slice: "Cognitive Protocol Mapping Closeout v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - protocol
  - cognitive-factory
  - architecture
  - index
  - closeout
related:
  - "02 Platform/architecture/cognitive-factory/protocol-node-specs.md"
  - "06 Execution/patterns/discern-station-application-mapping-v1.md"
  - "06 Execution/patterns/discern-micro-protocol-design-v1.md"
  - "06 Execution/patterns/decide-station-application-mapping-v1.md"
  - "06 Execution/patterns/synthesize-station-application-mapping-v1.md"
  - "02 Platform/architecture/current-agent-run-contract.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# Cognitive Protocol Mapping Closeout v1 -- Evidence Report

## purpose

Close the five-station Cognitive Protocol mapping thread (Perceive -> Interrogate -> Discern -> Decide ->
Synthesize) by reconciling the station docs into one index, verifying the proposed contract chain
(`DiscernAssessment` v2 -> `DecisionPosture` -> `SynthesizedArtifactPackage`), recording the responsibility
boundaries and runtime gaps, and defining the minimum-viable **read-only** implementation path. Docs-only: no
runtime, prompt, selection, confidence, buyer, metrics, or schema change, and no protocol runner implemented.

## context

Four design/mapping slices completed the thread: Discern Station Application Mapping v1, Discern Micro-Protocol
Design v1, Decide Station Application Mapping v1, Synthesize Station Application Mapping v1. Perceive and Interrogate
were already documented in `phases/*.md` and `protocol-node-specs.md`. This report consolidates all five, checks the
contract chain for drift, and states what may be built read-only later. `protocol-node-specs.md` remains the
canonical deep node index; this report is the thread-level closeout and cross-link, not a duplicate index.

## scope

**Included:** five-station inventory; the unified protocol chain; contract-chain consistency check;
responsibility-boundary table; runtime gap map; minimum-viable read-only implementation path; deferred runtime work.
**Excluded:** any runtime/code/prompt/selection/confidence/buyer/metrics/schema change; implementing any contract or
runner; altering the node specs or phase docs.

## five-station inventory (Phase 1)

| station | micro-actions | doctrine doc | mapping/design doc | runtime status | model involvement | current output | downstream consumer |
|---|---|---|---|---|---|---|---|
| Perceive | Detect, Frame, Aim | `phases/perceive.md` + node-specs | (node-specs) | implemented | hybrid: deterministic retrieval (`SportsRetriever`) + model narrative | `SportsRetrievalOutput`, `GroundedSignals`/`MissingSignals`/`SignalAvailability`, `perceive.*` | Interrogate, Discern |
| Interrogate | Question, Probe, Verify | `phases/interrogate.md` + node-specs | (node-specs) | implemented | hybrid: Question/Verify model; Probe deterministic (`BuildProbe`) | `interrogate.*`, `SignalFollowUps` | Discern |
| Discern | Weigh, Contrast, Stress | `phases/discern.md` + node-specs | [[discern-station-application-mapping-v1]], [[discern-micro-protocol-design-v1]] | implemented (Weigh backbone) + docs-only design (`DiscernAssessment` v2) | hybrid: Weigh deterministic backbone + model narrative; Contrast/Stress model today | `discern.*`, `SignalAvailability` grades | Decide |
| Decide | Resolve, Position, Justify | `phases/decide.md` + node-specs | [[decide-station-application-mapping-v1]] | implemented + docs-only design (`DecisionPosture`) | hybrid: model proposes; confidence deterministic (`SportsEvaluator`); posture clamped | `decide.*`, `AgentRun.LeanSide`, `AggregateConfidence`, `ConfidenceBand` | Synthesize |
| Synthesize | Integrate, Compose, Deliver | `phases/synthesize.md` + node-specs | [[synthesize-station-application-mapping-v1]] | implemented (deterministic) + docs-only design (`SynthesizedArtifactPackage`) | deterministic (`SportsComposer`, no model call); buyer copy distributed | `AgentRunExecutionResult`/`OutputJson`, `AgentRunResultDto` | buyer surface / `/dev/artifacts` / calibration |

## protocol chain summary (Phase 2)

- **Perceive** -- stages grounded context; surfaces what is, names what is missing, aims attention.
- **Interrogate** -- Question -> Probe -> Verify: names gaps, gathers/derives evidence, verifies enrichment.
- **Discern** -- Weigh -> Contrast -> Stress: grades, compares, and pressure-tests the enriched evidence.
- **Decide** -- Resolve -> Position -> Justify: commits to an allowed posture under deterministic blockers and clamps.
- **Synthesize** -- Integrate -> Compose -> Deliver: packages the posture safely for internal and buyer surfaces.

**Core operating phrase:** *Interrogate expands the evidence set; Discern judges the enriched evidence set; Decide
commits to posture; Synthesize packages posture safely.*

## contract chain consistency (Phase 3)

| contract | producer | consumer | det vs llm | buyer-visible | persisted today | future persistence |
|---|---|---|---|---|---|---|
| `DiscernAssessment` v2 | Discern | Decide | typed-deterministic findings + subordinate llm narrative | no | no (view over persisted fields) | read-only projection |
| `DecisionPosture` | Decide | Synthesize | model-proposed + deterministic clamps + deterministic confidence | leanSide/confidence yes; rest no | partially (`LeanSide`, confidence exist) | read-only projection |
| `SynthesizedArtifactPackage` | Synthesize | buyer surface / internal / calibration | deterministic assembly + rule-bounded llm copy | buyerSafe* yes; internal no | partially (artifact/DTO exist) | read-only projection |

**Chain carry-through (consistent):** `decisionStatus`, `leanSide`, `postureStrength`, `confidence`,
`confidenceBand`, `evidenceSufficiency` flow Discern->Decide->Synthesize with stable names. `decisionPressure`,
`abstentionPressure`, `needsHumanReview`, `confidenceCeilingAdvisory` appear in both `DiscernAssessment` and
`DecisionPosture` -- this is the **handoff** (Discern produces, Decide consumes), not duplication.
`primarySupportSignals`/`limitingFactors`/`acceptedContrasts`/`activeStressors`/`buyerSafetyConstraints` carry
Decide->Synthesize as intended.

**Boundary checks (all PASS in design):**

- **Confidence ownership:** deterministic (`SportsEvaluator`) at every stage; no contract recomputes it
  (`confidenceSource = deterministic_existing`). PASS.
- **Buyer-safe copy:** only at `SynthesizedArtifactPackage` (`buyerSafe*`); Decide emits only
  `buyerSafetyConstraints` (constraints, not copy). PASS.
- **Route/provenance leakage:** `routeProvenanceInternal` marked buyer-visible = no; provenance stays internal
  across all three. PASS.
- **Discern does not decide:** `DiscernAssessment` carries no `leanSide`/posture -- pressure only. PASS.
- **Synthesize does not intensify:** posture-strength ceiling rule, never upward. PASS.

**Open design questions (naming/field, non-blocking):**

1. `unsupportedClaims[]` exists on `DiscernAssessment` but is not explicitly carried on `DecisionPosture`/
   `SynthesizedArtifactPackage`. Decide and Synthesize each have an "unsupported-claim exclusion" rule, so the
   intent is *drop at each stage* -- but for audit continuity, consider carrying a single `unsupportedClaims`
   through the chain. **Recommend: carry-through for audit.**
2. `residueNotes` appears in all three contracts, station-scoped. Ambiguity risk when aggregated.
   **Recommend: stage-tag it** (e.g. `discernResidue` / `decideResidue` / `synthResidue`) or a `{stage, note}` shape.
3. `advertisedStrength` is produced by buyer projection and consumed by `SynthesizedArtifactPackage`, bypassing
   `DecisionPosture` (which only references it as a constraint). Correct today (buyer-projection-owned), but note
   the producer is outside the Decide contract.
4. `confidenceSource` is explicit on `DecisionPosture` but implicit on `SynthesizedArtifactPackage`.
   **Recommend: assert `confidenceSource` on the Synthesize package too** for symmetry.
5. `verifiedInterrogateAnswers[]` is consumed by Discern.Weigh and not carried downstream -- intended; noted for
   completeness.

No duplicate-field drift, no confidence-ownership violation, no buyer-boundary violation, no provenance-leak found.

## station responsibility boundary (Phase 4)

| responsibility | owning station | read-only stations | forbidden stations | current runtime owner | future canonical owner |
|---|---|---|---|---|---|
| source readiness | Perceive (pre) | Discern | Decide, Synthesize | `GET /source-readiness` (preflight) | Perceive |
| signal follow-up questions | Interrogate (Question) | Discern | Synthesize | model + `SignalFollowUpEvaluator` | Interrogate |
| probe results | Interrogate (Probe) | Discern | Decide, Synthesize | `BuildProbe` (deterministic) | Interrogate |
| verification status | Interrogate (Verify) | Discern | Synthesize | model (`interrogate.verify`) | Interrogate |
| evidence weighting | Discern (Weigh) | Decide | Synthesize | `SignalQualityEvaluator` + model narrative | Discern |
| contrast finding | Discern (Contrast) | Decide | Synthesize | model (`discern.contrast`) | Discern |
| stress/blocker finding | Discern (Stress) | Decide | Synthesize | model (`discern.stress`) | Discern |
| decision allowed / no-decision / needs-review | Decide (Resolve) | Synthesize | Discern, Synthesize (set) | `LeanSide` clamp (no-decision only today) | Decide |
| lean side | Decide (Position) | Synthesize | Discern, Synthesize | model -> FastAPI clamp (`AgentRun.LeanSide`) | Decide |
| posture strength | Decide (Position) | Synthesize | Synthesize (intensify) | model -> enum clamp + `block_aggressive_posture` | Decide |
| confidence | platform (`SportsEvaluator`) | Decide, Synthesize | model (owns) | `EvaluatorOutput.AggregateConfidence` | platform (deterministic) |
| justification | Decide (Justify) | Synthesize | Discern | model (`decide.justify`) | Decide |
| buyer-safe copy | Synthesize (Compose) | -- | Decide, Discern | FastAPI sanitizer + Angular mapper (distributed) | Synthesize |
| buyer delivery | Synthesize (Deliver) | -- | Decide | `AgentRunsController` + DTO omission | Synthesize |
| route / provenance | platform (route decision) | Discern, Decide | buyer surface | route decision -> `/rows`/provenance (internal) | platform (internal) |
| calibration outcome | platform (post-game) | all (read) | live pipeline (write) | `AgentRunOutcome` / `RunEvaluator` | platform (post-game) |

Enforced invariants: confidence is platform-owned/deterministic (not model); buyer-safe copy is Synthesize's, not
Decide's; registry routing/provenance stays internal; Discern does not decide; Synthesize does not intensify
posture; Interrogate names/probes/verifies gaps but does not grade final decision pressure (that is Discern).

## runtime gap map (Phase 5)

- **Already implemented:** Perceive retrieval + `perceive.*`; Interrogate Question/Verify + deterministic Probe;
  Discern.Weigh deterministic backbone; Decide posture clamp + `block_aggressive_posture` + deterministic
  confidence + `LeanSide` clamp; Synthesize deterministic compose/deliver; buyer field omission on the DTO.
- **Represented but distributed:** buyer-safe copy (FastAPI sanitizer + Angular buyer-signal mapper + DTO omission,
  not centralized in `SportsComposer`).
- **Docs-only (designed, not built):** `DiscernAssessment` v2, `DecisionPosture`, `SynthesizedArtifactPackage`;
  Discern.Contrast/Stress typed backbones.
- **Derivable from current fields:** all "now"-tagged deterministic rules across the three taxonomies (source
  completeness/reliability, market depth/agreement, regime attribution, route fallback, observed-vs-selected,
  identity safety, abstention pressure, needs-review trigger, buyer-safety propagation, field omission).
- **Future-only (needs new capture/state or wiring):** source freshness (not fully captured today); `needs_review`
  as a runtime state; fallback-ladder `ConfidencePermission`/`PosturePermission` enforcement (observational only
  today -- persisted, not consumed by `SportsEvaluator`).
- **Should remain deferred:** any confidence-formula wiring of `confidenceCeilingAdvisory`; any posture/abstention
  behavior change; centralizing buyer projection; a live protocol runner. All are behavior-changing and out of the
  read-only path.

## minimum viable read-only implementation path (Phase 6)

Read-only first; each step additive, no behavior change, separately approved:

1. **Read-only protocol inspection exporter/endpoint** -- surface the existing persisted protocol material in one
   place (like the artifact inspection endpoint / `byObservedRoute`). No new model call, no persisted column.
2. **`DiscernAssessment` view** -- derive typed `evidenceWeights`/`contrastFindings`/`stressFindings` from
   already-persisted fields (`SignalAvailability`, `SignalFollowUps`, route/regime attribution). Read-only.
3. **`DecisionPosture` view** -- derive from existing `LeanSide`/`AggregateConfidence`/`ConfidenceBand`/posture +
   the `DiscernAssessment` view. References confidence; never recomputes.
4. **`SynthesizedArtifactPackage` view** -- derive from the existing artifact + buyer-projection fields; keep
   internal vs buyer surfaces distinct; provenance internal.
5. **Surface only in dev/internal inspection** (`/dev/artifacts`-style). No buyer surface change.
6. **Tests** proving no prompt/model/selection/decision/confidence/buyer/metrics change: byte-identical model input;
   confidence unchanged; buyer DTO fields unchanged; provenance absent from buyer; `/metrics` denominator unchanged.
7. **Only later** consider runtime wiring (behavior-changing) -- a separate Evidence-Readiness-gated slice, not the
   immediate next step.

**Remains forbidden throughout:** changing confidence, posture, lean, buyer copy, prompt selection, registry
default, metrics denominator, or schema; enforcing the fallback ladder; enabling `needs_review` as a live state.

## safety / non-actions

Docs-only. **Not changed:** runtime code; prompt text; prompt-selection behavior; registry routing (stays
default-off); confidence formula; advertised strength; evidence sufficiency; buyer copy / buyer artifact output;
metrics denominator; DB schema. **Not done:** paid model calls (0); sports runs (0); reconciliation writes (0);
external sports API calls (0); DB migrations/backfills (0); no contract or runner implemented; no dormant runner
wired into production. Node specs and all five phase docs are referenced and unchanged. v8 generation budget
remains closed at 10.

## recommended next slice

If the pending v8 cohort games are **Final** on StatsAPI: **Settle-and-Reassess Registry-Routed v8 Cohort** (the
standing runtime priority) -- settle each via the reconciliation residue contract. If they are **not** Final:
**Cognitive Protocol Read-Only Inspection Plan v1** (spec step 1 of the read-only path above, still docs-only), or
pause protocol work until settlement finality. No behavior-changing implementation is recommended as the immediate
next step.

## related

- [[protocol-node-specs]] -- canonical deep node index (all 15 nodes); this report is the thread-level closeout.
- [[discern-station-application-mapping-v1]] / [[discern-micro-protocol-design-v1]] / [[decide-station-application-mapping-v1]] / [[synthesize-station-application-mapping-v1]] -- the four mapping/design docs consolidated here.
- `02 Platform/architecture/current-agent-run-contract.md` -- the persisted fields the read-only views would derive from.
