---
title: "Decide Station Application Mapping v1"
type: "execution-pattern"
date: "2026-07-04"
status: "draft"
project: "DAI"
slice: "Decide Station Application Mapping v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - protocol
  - decide
  - cognitive-factory
  - architecture
  - calibration
related:
  - "06 Execution/patterns/discern-micro-protocol-design-v1.md"
  - "06 Execution/patterns/discern-station-application-mapping-v1.md"
  - "02 Platform/architecture/cognitive-factory/protocol-node-specs.md"
  - "02 Platform/architecture/cognitive-factory/phases/decide.md"
  - "02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md"
  - "02 Platform/architecture/current-agent-run-contract.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# Decide Station Application Mapping v1

## purpose

Map DAI's current sports decision pipeline into the Cognitive Protocol's **Decide** station and its three
micro-actions -- Resolve, Position, Justify. Define what Decide consumes from `DiscernAssessment` v2, what it
produces, what residue it preserves, and how it hands forward to Synthesize. Design-only: no runtime, prompt,
selection, confidence, advertised-strength, evidence-sufficiency, buyer, metrics, or schema change.

## context

Decide is the fourth station: Perceive -> Interrogate -> Discern -> **Decide** -> Synthesize. The prior two slices
matured Discern into a micro-protocol that produces `DiscernAssessment` v2 -- typed `evidenceWeights`,
`contrastFindings`, `stressFindings`, `decisionPressure`, `abstentionPressure`, `needsHumanReview`,
`confidenceCeilingAdvisory`, `unsupportedClaims`, `unresolvedQuestions`, `verifiedInterrogateAnswers`. Inherited
principle: **Discern does not decide; it prepares the evidence judgment Decide consumes.** This slice maps that
handoff onto Resolve/Position/Justify and refines the `DecisionPosture` output contract, honoring the fact that
today's final confidence is deterministic (`SportsEvaluator`) and posture is a clamped enum.

The canonical node contracts in `protocol-node-specs.md` and `phases/decide.md` are referenced and unchanged.

## scope

**Included:** a Decide mental model; the Decide doctrine + current-pipeline decision field map; Resolve/Position/
Justify operational mapping (inputs from DiscernAssessment v2, current runtime analogs, deterministic rules, LLM
role, outputs, residue, forbidden); a proposed `DecisionPosture` contract; a deterministic rule taxonomy with
current-vs-future implementability; the LLM role boundary; the Synthesize handoff.

**Excluded:** any runtime/code/prompt/selection/confidence/advertised-strength/evidence-sufficiency/buyer/metrics/
schema change; implementing `DecisionPosture` or a Decide runner; a full Synthesize design; enabling registry
routing; altering the node specs or the confidence formula.

## decide mental model

*Decide sets the read stance and the calibration narrative that justifies it* -- **after** Discern, **before**
Synthesize. Three honest truths about Decide as it exists today:

- **Final confidence belongs to the platform, not the model.** `SportsEvaluator.Evaluate` produces
  `AggregateConfidence` with grounding-tier clamps (zero grounded: dampen 0.75, clamp [0.30,0.60]; partial: dampen
  0.90, clamp [0.35,0.75]; full: clamp [0.35,0.85]). The model's local `AnalyzerConfidence` is stored separately for
  the learning loop and is never the user-facing number.
- **Posture is a clamped enum, not a pick.** The model proposes; FastAPI validates/clamps to `{play, pass, monitor,
  wait, compare, avoid}`; `block_aggressive_posture` downgrades an aggressive `play`. The UI labels it **Read
  Stance**, never "Pick".
- **Decide commits an *internal* stance; it does not write buyer copy.** Buyer-safe language is Synthesize's job.
  Decide emits constraints/caveats (`buyerSafetyConstraints`) for Synthesize to honor, not marketing text.

Decide consumes `DiscernAssessment`; it must not re-derive Discern's judgment, recompute confidence, or invent
support Discern did not ground.

## decide doctrine inventory (Phase 1)

| path | purpose | status | relevance to R/P/J | touched |
|---|---|---|---|---|
| `phases/decide.md` | Decide phase doctrine; legacy→canonical map; clamps | v1 doctrine | high (all three) | no |
| `protocol-node-specs.md` (Decide) | 11-facet Resolve/Position/Justify node contracts | doctrine | high | no |
| `CognitiveProtocolBuilder.cs` | passes seed `decide.resolve/position/justify` through 1:1; preserves validated posture string | implemented | Position/Resolve/Justify representation | no |
| `SportsEvaluator` / `EvaluatorOutput.AggregateConfidence` | deterministic final confidence + band + clamps | implemented | Justify (confidence), Position (band) | no |
| FastAPI posture clamp + `LeanSide` clamp | enum/direction validation to null | implemented | Position, Resolve | no |
| `signal-fallback-ladder.md` (`ConfidencePermission`/`PosturePermission`) | per-fallback confidence/posture permissions | **observational v1** (persisted, NOT yet consumed by SportsEvaluator) | future Position/Justify clamps | no |
| `current-agent-run-contract.md` | persisted decision fields (`LeanSide`, OutputJson, confidence) | implemented | field map | no |
| buyer artifact projection / advertised strength | buyer-visible surface derived downstream | implemented | Synthesize-owned; Decide references as constraint | no |

## current pipeline decision field map (Phase 2)

`stage` = pre-model | model-authored | det-post (deterministic post-model) | buyer-projection | post-game.
`D` = Decide relevance; `S` = Synthesize relevance; `action` = Decide should read / write / preserve.

| field | producer | stage | persist | buyer | D | S | action |
|---|---|---|---|---|---|---|---|
| lean (text) | model (`decide.resolve`/voice) | model-authored | yes | yes | writes | preserves | write |
| leanSide | model → FastAPI clamp | model + det | yes (`AgentRun.LeanSide`) | via lean | writes | preserves | write |
| confidence (`AggregateConfidence`) | `SportsEvaluator` | det-post | yes | yes | reference | preserves | **reference (never recompute)** |
| confidenceBand | `SportsEvaluator` | det-post | yes | ish | reference | preserves | reference |
| AnalyzerConfidence | model | model-authored | yes | no | reference (learning) | n/a | preserve |
| advertisedStrength | buyer projection | buyer-projection | yes/projected | **yes** | reference (constraint) | **owns** | preserve (never change) |
| evidenceRichness / evidence sufficiency | retriever / sufficiency gate | det-post / pre | yes | influences | reads (clamp input) | preserves | read |
| source sufficiency | `SignalAvailability` | pre | yes | no | reads (Resolve) | n/a | read |
| block_aggressive_posture | `SignalQualityEvaluator` | pre | yes | no | reads (Position clamp) | n/a | read |
| no-decision / null lean | Decide.Resolve | model + det | yes (`leanSide` null) | ish | writes | preserves | write |
| rationale (`decide.justify`/calibrate) | model | model-authored | yes | partial | writes | composes | write |
| summary | model + composer | model + compose | yes | **yes** | contributes | **owns** | preserve |
| key factors | model | model-authored | yes | yes | reference | composes | reference |
| buyer-safe lean language | buyer projection / Synthesize | buyer-projection | yes/projected | **yes** | **not owner** | **owns** | must-not-author |
| market consensus side | market context | pre | yes | no | reads (Position align/conflict) | n/a | read |
| market agreement | derived | pre | partial | no | reads | n/a | read |
| observedDataRegime | `observed_data_regime` | det (at-run) | yes | no | reads (provenance) | preserves | read |
| selectedDataRegime | registry selection | at-run | yes (null live) | no | reads | preserves | read |
| promptSource | route decision | at-run | yes | no | reads (provenance) | preserves | read |
| selectedPromptPath | route decision | at-run | yes | no | reads | preserves | read |
| recipeId / version / hash | registry selection | at-run | yes (null live) | no | reads | preserves | read |
| fallbackReason | route decision | at-run | yes | no | reads (route-fallback clamp) | preserves | read |
| outcome / evaluation | `AgentRunOutcome` / `RunEvaluator` | post-game | yes | no | post-game calibration only | n/a | preserve (post-game) |

## decide station mapping (Phase 4)

Expected posture: **Resolve deterministic-first; Position mixed (model proposes, deterministic blockers/clamps
constrain); Justify LLM-authored but grounded in typed findings; buyer-safe copy belongs to Synthesize.**

### Resolve

- **purpose:** determine whether the run may proceed to a posture, must abstain (no-decision), or needs human
  review. Answers: is a decision allowed? is abstention required? is review required? are deterministic blockers
  present? is evidence sufficient for a lean? is aggressive posture blocked? is provenance trustworthy enough?
- **inputs from DiscernAssessment v2:** `stressFindings[].blocksDecision`, `needsHumanReview`, `abstentionPressure`,
  `decisionPressure.decisionReadiness`, `unresolvedQuestions`, plus `block_aggressive_posture`.
- **current runtime analogs:** legacy `decide.voice` (model framing); FastAPI `LeanSide` clamp to null (the current
  no-decision mechanism).
- **deterministic rules:** identity-safety blocker, source-sufficiency blocker, missing-critical-signal blocker,
  abstention-pressure threshold, needs-human-review threshold, no-decision/null-lean rule (see taxonomy). Any
  `blocksDecision` stressor forces `no_decision` or `needs_review`.
- **optional LLM role:** frame the resolved direction/non-direction without hype; it does not choose whether a
  blocker applies.
- **outputs:** `decisionStatus` (decision | no_decision | needs_review), `resolvedBy`.
- **residue:** which blocker/pressure drove the status; the unresolved questions carried forward.
- **forbidden:** proceed past a deterministic blocker; treat thin/blocked evidence as decision-ready; let the model
  clear a blocker.

### Position

- **purpose:** commit the internal decision posture **if Resolve permits**. Answers: home/away/null lean? posture
  strength? support level? constraints? clamped by evidence sufficiency? `block_aggressive_posture`? align/conflict
  with market consensus?
- **inputs from Resolve + DiscernAssessment v2:** `decisionStatus`; `decisionPressure.leanSupport`/
  `abstentionSupport`; `evidenceWeights`; `contrastFindings` (market alignment); `block_aggressive_posture`;
  `confidenceCeilingAdvisory`.
- **current runtime analogs:** legacy `decide.posture` (model proposes) + FastAPI enum clamp `{play, pass, monitor,
  wait, compare, avoid}` + `block_aggressive_posture` downgrade + `LeanSide` extraction/clamp. **These clamps are
  implemented today.**
- **deterministic rules:** `block_aggressive_posture` clamp, route-fallback/provenance clamp, observed-vs-selected
  regime-mismatch clamp, confidence/evidence-sufficiency clamp, no-decision/null-lean rule. Model proposes a
  posture; the deterministic clamps are authoritative.
- **optional LLM role:** propose the lean/posture from evidence; the platform validates and clamps it.
- **outputs:** `leanSide` (home | away | null), `postureStrength` (none | slight | measured | strong),
  `aggressivePostureBlocked`.
- **residue:** the proposed-vs-clamped posture, which clamp fired, market alignment relationship.
- **forbidden:** emit a posture outside the enum; keep `play` when a fallback/`block_aggressive_posture` blocks it;
  inflate `postureStrength` beyond evidence sufficiency; label posture a "Pick" or a lock.

### Justify

- **purpose:** produce the evidence-backed rationale for the chosen posture, preserve constraints/caveats, and hand
  Synthesize what it must not overstate. Answers: why this position? what evidence carried most weight? what
  conflicts were acknowledged? what stressors constrained it? what remained unresolved? what caveats persist?
- **inputs from Position + Resolve + DiscernAssessment v2:** `evidenceWeights`, `contrastFindings`, `stressFindings`,
  `unresolvedQuestions`, `verifiedInterrogateAnswers`, `confidenceCeilingAdvisory`, the deterministic
  `Confidence`/`ConfidenceBand`.
- **current runtime analogs:** legacy `decide.calibrate` (model rationale **sentence only**); `SportsEvaluator`
  owns the confidence number and band.
- **deterministic rules:** confidence/evidence-sufficiency consistency check (reference only, never recompute);
  buyer-safety constraint propagation; calibration-insufficiency warning; unsupported-claim exclusion.
- **optional LLM role:** author the rationale language, grounded in the typed findings; each claim must map to an
  evidence weight, contrast, stressor, or verified Interrogate answer.
- **outputs:** `justificationSummary`, `primarySupportSignals[]`, `limitingFactors[]`, `acceptedContrasts[]`,
  `activeStressors[]`, `buyerSafetyConstraints[]`.
- **residue:** the full rationale trace + constraints Synthesize must honor.
- **forbidden:** let stated confidence exceed the deterministic value; invent rationale not traceable to a finding;
  author buyer marketing copy; restate the number instead of justifying it.

**Decide's boundary in one line:** it commits an internal, clamped, justified stance; it does not compute the
confidence number, does not write buyer copy, and does not re-judge the evidence.

## proposed DecisionPosture contract (Phase 3)

A read-model that captures Decide's committed stance. **Docs-only; not implemented; not runtime.** It references the
deterministic confidence, preserves Discern's blockers, and recomputes nothing. `det` = deterministic, `llm` =
LLM-authored (grounded).

| field | type | producer | consumer | det/llm | persisted | buyer-visible |
|---|---|---|---|---|---|---|
| agentRunId / sourceProvider / externalGameId | ids | platform | audit/Synthesize | det | future | no |
| decisionStatus | decision \| no_decision \| needs_review | Resolve | Synthesize | det | future | no |
| resolvedBy | deterministic_blocker \| evidence_supported \| abstention_pressure \| model_position \| manual_review_required | Resolve | audit | det | future | no |
| leanSide | home \| away \| null | Position | Synthesize | model + det clamp | yes (today `AgentRun.LeanSide`) | via lean |
| postureStrength | none \| slight \| measured \| strong | Position | Synthesize | model + det clamp | future | internal |
| confidence | number | `SportsEvaluator` | Synthesize | **det (referenced)** | yes (today) | yes |
| confidenceBand | high \| medium \| low | `SportsEvaluator` | Synthesize | det | yes | ish |
| confidenceSource | `deterministic_existing` | contract constant | audit | det | future | no |
| confidenceCeilingAdvisory | number? | Discern | Justify | det, **advisory only** | future | no |
| evidenceSufficiency | enum/int | sufficiency gate | Position/Justify | det | yes | influences |
| aggressivePostureBlocked | bool | `block_aggressive_posture` | Position | det | yes | no |
| decisionPressure | `{leanSupport, abstentionSupport, uncertaintyDrivers, decisionReadiness}` | Discern | Position | det | future | no |
| abstentionPressure | low \| med \| high | Discern | Resolve | det | future | no |
| needsHumanReview | bool | Discern | Resolve | det | future | no |
| primarySupportSignals[] | string[] | Justify (from evidenceWeights) | Synthesize | det + llm | future | internal |
| limitingFactors[] | string[] | Justify (from stressFindings) | Synthesize | det + llm | future | internal |
| acceptedContrasts[] | ContrastFinding[] | Justify (from contrastFindings) | Synthesize | det | future | no |
| activeStressors[] | StressFinding[] | Justify (from stressFindings) | Synthesize | det | future | no |
| unresolvedQuestions[] | string[] | Discern (Interrogate residue) | Synthesize | det | future | no |
| justificationSummary | string | Justify | Synthesize | llm (grounded) | future | no |
| buyerSafetyConstraints[] | string[] | Justify | **Synthesize (honors)** | det + llm | future | drives buyer |
| residueNotes | string | all | audit/calibration | det | future | no |

**Guarantees:** does not redefine the confidence formula; references confidence but never recomputes it
(`confidenceSource = deterministic_existing`); preserves Discern's deterministic blockers; does not promote buyer
copy (only constraints for Synthesize). It is a superset **view**; it does not replace `AgentRun.LeanSide`,
`OutputJson`, or the canonical `CognitiveProtocol.Decide` block.

## deterministic decide rule taxonomy (Phase 5)

`blocks` = can set no_decision/needs_review; `clamps` = can constrain posture/confidence. Implementability: **impl**
= enforced in code today; **deriv** = computable from persisted fields but not yet enforced; **future** = needs new
capture/wiring.

| category | signal read | rule type | output | blocks | clamps | feeds | impl |
|---|---|---|---|---|---|---|---|
| identity-safety blocker | resolved identity / `externalGameId` | ambiguity gate | safe/ambiguous | yes | no | Resolve | deriv |
| source-sufficiency blocker | `GroundedSignals`/richness vs floor | threshold | ok/below-floor | yes | no | Resolve | deriv (sufficiency gate exists) |
| missing-critical-signal blocker | `MissingSignals` primaries | presence | ok/missing-primary | context | no | Resolve | deriv |
| block_aggressive_posture clamp | `ConfidenceEffect` | posture downgrade | play→lower | no | yes | Position | **impl** |
| route-fallback / provenance clamp | `fallbackReason`/`selectedPromptPath` | fallback detect | none/fallback | no | yes | Position/Resolve | deriv (not enforced) |
| observed-vs-selected regime clamp | `observedDataRegime` vs `selectedDataRegime` | null-aware match | match/mismatch/na-live | no | yes | Position | deriv |
| confidence/evidence-sufficiency clamp | `Confidence` vs `EvidenceRichness` | consistency check | consistent/over-confident | no | reference | Justify/Position | **impl** (calibration clamps + `confidence_high_for_partial_evidence` flag) |
| abstention-pressure threshold | `abstentionPressure` | threshold | proceed/abstain-pressure | no | no | Resolve | deriv/future |
| needs-human-review threshold | `needsHumanReview` | any-of trigger | true/false | routes | no | Resolve | deriv/future |
| no-decision / null-lean rule | `leanSide` null / split signals | null rule | decision/no_decision | yes | no | Resolve/Position | **impl** (`LeanSide` clamp) |
| buyer-safety constraint propagation | advertised strength / lean vs evidence | overclaim detect | constraint list | no | no | Justify→Synthesize | deriv |
| calibration-insufficiency warning | pooled bucket / route-level n | min-sample | sufficient/thin | conclusions | no | Justify | deriv |
| unsupported-claim exclusion | LLM claims vs typed findings | grounding filter | excluded → unsupportedClaims | no | no | Justify | deriv |
| fallback-ladder confidence/posture permission | `ConfidencePermission`/`PosturePermission` | permission clamp | preserved…reduced / aggressive_blocked | no | yes | Position/Justify | **observational v1** (persisted, NOT enforced by SportsEvaluator; future outcome-scoped wiring) |

Honest note: the confidence clamps and posture/`block_aggressive_posture` clamps are enforced today; the
fallback-ladder permissions are persisted but **not** yet wired into `SportsEvaluator` -- Decide reads them via the
diagnostics skill, not deterministic enforcement. Wiring them is a future, outcome-scoped, separately-approved
slice.

## LLM role boundary (Phase 6)

**Allowed:** propose the internal lean/posture from evidence; draft justification language; explain why key
evidence supports or limits the posture; summarize contrasts/stressors; identify rationale gaps for Synthesize to
preserve.

**Forbidden:** override identity safety, deterministic blockers, `block_aggressive_posture`, or route-fallback/
provenance clamps; change the confidence formula; inflate posture beyond evidence sufficiency; convert missing
evidence into support; invent rationale not traceable to a finding; produce buyer-safe marketing language directly;
promote/enable registry routing; alter calibration conclusions.

**Grounding requirements:** every justification claim maps to an evidence weight, contrast, stressor, or verified
Interrogate answer; unsupported claims are excluded / sent to `unsupportedClaims[]`; **deterministic blockers win**;
the confidence number stays deterministic; buyer-copy transformation is deferred to Synthesize.

## handoff to synthesize (Phase 7)

**Synthesize receives** (via `DecisionPosture`): `decisionStatus`, `leanSide`, `postureStrength`, `confidence` +
`confidenceBand`, `evidenceSufficiency`, `primarySupportSignals` (accepted support), `limitingFactors` (caveats),
`acceptedContrasts`/`activeStressors`, `buyerSafetyConstraints`, `justificationSummary`, route/provenance metadata
(`observedDataRegime`/`selectedPromptPath`/`recipe*`), and `residueNotes`.

**Synthesize must not receive:** raw unverified model speculation; unsupported claims (stay in
`unsupportedClaims`); hidden blockers (every blocker is a typed, named finding); unconstrained marketing language
(Decide emits constraints, not copy); calibration conclusions not backed by gates.

**How Synthesize's micro-actions will use Decide (full design deferred to a later slice):**

- **Integrate** consumes posture + route + evidence + caveats, adding no new claim.
- **Compose** builds the internal artifact and the buyer-safe surface, honoring `buyerSafetyConstraints` (the buyer
  transformation Decide deliberately did not perform).
- **Deliver** returns the correct output surface (internal vs buyer) while preserving the boundary -- posture
  labeled **Read Stance**, never "Pick"; no leak of raw prompts/provenance internals to the buyer.

## safety / non-actions

Docs-only. **Not changed:** runtime code; prompt text; prompt-selection behavior; registry routing (stays
default-off); confidence formula; advertised strength; evidence sufficiency; buyer copy / buyer artifact output;
metrics denominator; DB schema. **Not done:** paid model calls (0); sports runs (0); reconciliation writes (0);
external sports API calls (0); DB migrations/backfills (0); no `DecisionPosture` or Decide runner implemented; no
dormant runner wired into production. Node specs, `phases/decide.md`, and `SportsEvaluator` are referenced and
unchanged. v8 generation budget remains closed at 10.

## next recommended slice

If the pending v8 cohort games are **Final** on StatsAPI: **Settle-and-Reassess Registry-Routed v8 Cohort** (the
standing runtime priority) -- settle each via the reconciliation residue contract. If they are **not** Final:
**Synthesize Station Application Mapping v1** -- apply the same treatment to Synthesize (Integrate / Compose /
Deliver) consuming this `DecisionPosture`, completing the five-station protocol-mapping thread while settlement
waits on finality.

## related

- [[discern-micro-protocol-design-v1]] -- the `DiscernAssessment` v2 this station consumes.
- [[protocol-node-specs]] -- authoritative per-node Decide contracts, unchanged here.
- `02 Platform/architecture/cognitive-factory/phases/decide.md` -- Decide phase doctrine + clamps.
- `02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md` -- the observational-v1 permissions Decide will one day enforce.
- `02 Platform/architecture/current-agent-run-contract.md` -- persisted `LeanSide` / confidence fields.
