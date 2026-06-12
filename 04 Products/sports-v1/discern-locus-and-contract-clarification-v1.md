# Discern Locus and Contract Clarification v1

**date:** 2026-06-12
**status:** docs / contract only. no runtime, test, prompt, schema, buyer, confidence/posture/lean, evaluator, matcher, or reconciliation change. follows Discern Improvement Review v1 (vault commit 93a6f81).
**scope:** fix where Discern lives across the codebase and doctrine, and define -- on paper only -- the internal contract Discern should hand to Decide. No type is implemented in code. Internal Calibration Read Surface v1 is explicitly not built here.

## 1. Executive summary

Discern remains conceptually central: it is the stage that decides what counts as evidence and how much weight each signal carries. But its *behavior must not change* until reconciled-outcome evidence exists (deferred ledger entries 12, 25 -- all four Stage 0 candidates are still pre-settlement). Discern grades evidence weight, and weight grading can only be validated against outcomes; tuning it now would be intuition over evidence.

This document does two safe things. It maps the five+1 places "Discern" currently lives so no future builder mistakes the dormant `DiscernStationRunner` for the live surface. And it writes down the Discern -> Decide handoff as a named contract shape -- a paper artifact that consolidates concepts that *already flow informally today* (`block_aggressive_posture`, the fallback ladder's `ConfidencePermission` / `PosturePermission`, `ConfidenceBand`, follow-up records) so a future dormant code type has a target. Nothing is wired, activated, or tuned.

## 2. Current Discern locus map

| layer | where it lives | what it is | mutability today |
|---|---|---|---|
| model-emitted protocol prose | analyzer `discern` block -> `DiscernProtocol.Weigh/Contrast/Stress` (`CognitiveProtocol.cs`) | three nullable sentences the model authors; registry marks Discern stations `ExecutionOwner.Model`; platform passes them through unchanged via `CognitiveProtocolBuilder` | live every run, model-owned, pass-through |
| deterministic signal/evidence grading | `SignalQualityEvaluator`, `SignalFollowUpEvaluator` (+ six-tier sharp/public fallback ladder), `SportsEvaluator` band | the real structured judgment doctrine assigns to Discern.Weigh, but it sits under signal/retrieval plumbing, not a "Discern" namespace | live every run, platform-owned, deterministic |
| artifact fields that approximate Discern | `signalAvailability` (quality / decision-use / confidence-effect), `signalFollowUps` (status / FallbackType / Equivalence / ConfidencePermission / PosturePermission), `DiscernProtocol.*` prose | persisted expression of the grading + narrative | persisted, read-only after write |
| buyer-visible projection | `watch_for` <- `protocol.discern.stress` (`sports_analyzer.py`), rendered "What Could Change the Read"; `DiscernProtocolView` (Weigh/Contrast/Stress) via `ProtocolVocabularyMapper` / `CognitiveProtocolView` | the one buyer-facing Discern output is the stress line | live, buyer-facing, locked copy |
| dormant scaffolding | `DiscernStationRunner` / `IDiscernStationRunner` / `DiscernStationExecutionRequest` / `DiscernStationExecutionResult` / `DiscernStationAssessment` / `DiscernStationAssessmentStatus` (`DiscernStationRunner.cs`); plus probe-refresh-specific `ProbeRefreshDiscernReweigh` | generic runner validates input *shape* only and returns "input available for assessment" -- it does not weigh, contrast, or stress; the reweigh classifies a single refreshed signal as SupportsRead/WeakensRead/Neutral/NeedsHumanReview | dormant: no DI registration, no `ProtocolNodeRunner` wiring, no production caller (ledger 16) |
| docs-only doctrine | `phases/discern.md`, `protocol-vocabulary-map.md`, `signal-fallback-ladder.md` | canonical Weigh/Contrast/Stress definitions + legacy-to-canonical field mapping | doctrine, no behavior |

**The central fact:** the canonical Discern micro-actions and the code do not line up by name. Doctrine puts Stress under Discern; code emits it under `interrogate.stress`. The deterministic Discern surface lives under signal plumbing. The thing literally named `DiscernStationRunner` does the least actual discerning. A builder reading only the code would misread where Discern lives -- which is exactly what this map prevents.

## 3. Weigh / Contrast / Stress ownership

### Weigh
- **current source of truth:** split. Deterministic: `SignalQualityEvaluator` (quality `strong/usable/unavailable`, decision use, confidence effect) + `SportsEvaluator` (grounded count + flags -> `ConfidenceBand`). Qualitative: model `discern.weigh` sentence.
- **current runtime behavior:** runs every sports run inside `SportsRetrievalOutput`; `SignalFollowUpEvaluator` runs immediately after `SignalQualityEvaluator` with no model call between them.
- **current artifact expression:** `signalAvailability[*]` (quality/decisionUse/confidenceEffect), `signalFollowUps[*]`, `DiscernProtocol.Weigh` prose.
- **gaps:** the deterministic surface is not namespaced or characterized as "Discern"; a doctrinal rename could silently change behavior with no test guarding it as Discern.Weigh.
- **what should remain deferred:** any rename/move of the evaluators; any change to the grading rules or band clamps.

### Contrast
- **current source of truth:** model `discern.contrast` sentence only (market / sharp-public / external interpretation, or null when no market data).
- **current runtime behavior:** narrative only; there is no deterministic detector of two grounded signals disagreeing. The dormant `ProbeRefreshDiscernReweigh` is the nearest structured thing (single-signal effect classification), not a cross-signal comparator.
- **current artifact expression:** `DiscernProtocol.Contrast` prose.
- **gaps:** Contrast is the weakest micro-action; it cannot be calibrated because it produces no structured divergence output. Mitigated today because `sharp_public` (the main contrast input) is the dominant *missing* signal (Buyer Artifact UX Calibration v1).
- **what should remain deferred:** a deterministic Contrast helper -- speculative until contrast inputs are reliably grounded.

### Stress
- **current source of truth:** model `discern.stress` sentence. Single-source by canonical contract (Stress Collapse v1 retired the legacy `interrogate.stress` / `discern.test` duplication; the field still physically emits under `interrogate.stress` pending a future move slice).
- **current runtime behavior:** model-authored; structurally backstopped by `block_aggressive_posture` propagation and the evidence-sufficiency band gate, but the narrative itself is unvalidated against outcomes.
- **current artifact expression:** `DiscernProtocol.Stress` prose; buyer `watch_for` ("What Could Change the Read").
- **gaps:** buyer-visible but unmeasured -- ships straight from a model sentence with no outcome feedback. Honest framing mitigates; quality is unknown until reconciliation.
- **what should remain deferred:** moving Stress out of `interrogate.stress` into a canonical `discern.stress` runtime slot (touches prompt, parser, persisted shape, and the `watch_for` source -- out of scope); any tuning of the stress narrative.

## 4. Discern-to-Decide contract (proposed, docs-only, not wired)

A named handoff shape Discern *would* produce for Decide. This is a paper contract: **do not implement the type in code in this slice.** Each field is grounded in something that already exists or already flows informally today, so the future dormant type is consolidation, not invention.

```
DiscernAssessment (proposed; not implemented)
  weightedSignals[]            // per-signal: key, quality (strong/usable/unavailable),
                               //   decisionUse (primary/confirmation/proxy_candidate/directional_only),
                               //   confidenceEffect. SOURCE TODAY: SignalQualityEvaluator.
  dominantSignals[]            // the grounded signals carrying the read. derived from weightedSignals.
  opposingSignals[]            // grounded signals pulling against the lean (contrast). SOURCE TODAY:
                               //   none structured -- only discern.contrast prose. this field is the
                               //   contrast gap made explicit.
  unresolvedConflicts[]        // opposing pairs with no resolution; today implicit in prose only.
  evidenceSufficiencyBand      // thin/medium/rich. SOURCE TODAY: evidence-richness used by the
                               //   buyer band gate; kept distinct from numeric confidence.
  fragilityNotes[]             // structured form of discern.stress; the condition(s) under which
                               //   the read fails. SOURCE TODAY: discern.stress prose + watch_for.
  stressors[]                  // named risk factors (large spread, missing sharp_public, equal rest,
                               //   missing injury data). SOURCE TODAY: prompt fragility examples.
  postureConstraints[]         // strongest allowed posture. SOURCE TODAY: block_aggressive_posture +
                               //   fallback ladder PosturePermission (aggressive_allowed /
                               //   _if_corroborated / _blocked).
  confidenceConstraints[]      // ceiling on contributed confidence. SOURCE TODAY: fallback ladder
                               //   ConfidencePermission (confidence_preserved ... _reduced).
  recommendedDecisionGuardrails// derived advisory: e.g. "no high band with one grounded signal",
                               //   "play blocked". Decide remains free to apply; Discern only recommends.
  calibrationFeedbackHooks     // identifiers (run identity, signal keys, lean) that let a future
                               //   reconciliation loop attribute outcome back to weighed signals.
                               //   SOURCE TODAY: AgentRun identity columns + RunEvaluator lean-vs-result.
```

Contract intent: Discern emits *grading, conflict, fragility, and constraints*; it never emits a stance, a posture choice, a lean, or a final confidence number. Those belong to Decide (section 6). The contract is observational/advisory in spirit -- it mirrors how the fallback-ladder permissions are already "observational in v1" per `phases/decide.md`.

## 5. Boundary with Interrogate

Interrogate owns **Question, Probe, Verify**:
- **Question:** the strongest open question or counter-case against the emerging read (legacy `interrogate.balance`).
- **Probe:** targeted investigation / follow-up *requests* for signal gaps -- it *names candidates*, it does not fetch. Probe is deterministic platform output today (`SignalFollowUpEvaluator` names follow-up candidates; the model-emitted Probe field does not exist yet).
- **Verify:** confirm or reject claims against staged evidence and guardrails (legacy `interrogate.reframe`).
- plus **source/provenance readiness:** what is missing, what was probed, what was verified, and (once it exists) availability-fidelity / source-kind metadata.

Doctrinal line (from `phases/interrogate.md`): **"Interrogate names candidates. Discern grades them."** Interrogate must not classify a follow-up candidate against the fallback ladder; that grading is Discern's. **Discern should not own collection or source refresh** -- it never triggers a Perceive refresh, never calls a retrieve tool, and never decides what to fetch. Discern receives verified signals, missing-signal flags, fallback/proxy candidate *names*, and provenance readiness from Interrogate, then weighs/contrasts/stresses them.

## 6. Boundary with Decide

Decide owns **Resolve, Position, Justify**:
- **Resolve:** reconcile the weighed evidence into a single direction or non-direction (legacy `decide.voice`).
- **Position:** the decision posture from the niche vocabulary `play / pass / monitor / wait / compare / avoid` (legacy `decide.posture`; UI label `Read Stance`, never `Pick`).
- **Justify:** one-sentence rationale for why the stated confidence fits the evidence (legacy `decide.calibrate`).
- plus the **final calibrated confidence** (`AggregateConfidence` / `ConfidenceBand`) -- platform-owned via `SportsEvaluator`, not the model's local estimate.

**Discern should not choose the final stance.** It constrains and informs: it hands Decide the weighted signals, conflicts, fragility, and the posture/confidence *constraints* (the contract in section 4). Decide then resolves the lean, sets posture (honoring `block_aggressive_posture` and, in a future slice, `PosturePermission` / `ConfidencePermission`), and produces the final calibrated confidence. The split is: **Discern says how much the evidence is worth and where it is fragile; Decide says what stance that justifies.**

## 7. Calibration feedback plan

How future reconciled outcomes should inform Discern -- *later*, once real evaluations exist (gated by ledger 12, 25). This slice tunes nothing.

1. **Which weighed signals mattered.** Join reconciled `AgentRunEvaluation` (lean-vs-result) back to the run's `weightedSignals` to see whether grounded "strong/primary" signals actually tracked correct reads, or whether a particular signal class was noise.
2. **Recurring false confidence.** Find runs where Discern's grading supported a high band but the read was wrong -- recurring patterns point to a Weigh rule that over-credits a signal.
3. **Fragile reads that failed.** Check whether `fragilityNotes` / `stressors` predicted the actual failure mode (e.g. the named "missing sharp_public" risk materializing), measuring Stress quality.
4. **Contrast misses.** Detect cases where an `opposingSignal` was present but unweighted and the opposing side won -- evidence a deterministic Contrast helper would have caught.
5. **Inform future rules/tests/contracts.** Feed findings into `SignalQualityEvaluator` clamp adjustments, new quality rules, or the dormant assessment type -- each as its own evidence-gated slice with its own review. No confidence or behavior tuning happens without reconciled-outcome evidence.

## 8. Deferred implementation decisions

Explicitly deferred (no work in this slice; see ledger entry 16):

- activating `DiscernStationRunner` (no production caller; ledger 16/17/18)
- a deterministic Contrast helper
- implementing the section-4 `DiscernAssessment` type in code
- analyzer prompt changes / moving Stress out of `interrogate.stress`
- model-call expansion
- buyer copy changes ("What Could Change the Read" stays locked)
- confidence / posture / lean tuning; evaluator (`SportsEvaluator`/`SignalQualityEvaluator`/`SignalFollowUpEvaluator`) changes
- wiring `ConfidencePermission` / `PosturePermission` into the calibration path (still observational per `phases/decide.md`)
- generalizing `ProbeRefreshDiscernReweigh` (ledger 14)
- Internal Calibration Read Surface v1; dashboards
- outcome reconciliation / matcher / candidate changes

## 9. Recommended future Discern slices (at most two)

1. **Discern Assessment Contract v1 (code type only, dormant).** Implement the section-4 `DiscernAssessment` as an inert value-object type in `DevCore.Api.Protocols` -- no producer, no consumer, no wiring, no model call -- mirroring how `DiscernStationRunner` groundwork shipped dormant. Gives a future Discern->Decide handoff a concrete shape without activating behavior. Only worth doing if a second consumer or a concrete producer is on the horizon; otherwise stay docs-only (this document) to avoid a speculative one-use type.
2. **Discern Contrast Characterization v1 (tests/design only, AFTER real evaluations exist).** Once reconciled outcomes exist, characterize how often a structured Contrast would have changed a read (calibration finding #4), as a design + characterization-test exercise -- before any deterministic Contrast helper is built. Evidence-gated; not now.

## References

- `04 Products/sports-v1/discern-improvement-review-v1.md` (the review this clarifies)
- `02 Platform/architecture/cognitive-factory/phases/discern.md`, `interrogate.md`, `decide.md`
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md`, `signal-fallback-ladder.md`
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` (entries 12, 14, 16, 25)
- code: `DevCore.Api/AgentRuns/CognitiveProtocol.cs`, `ProtocolVocabularyMapper.cs`, `CognitiveProtocolView.cs`; `DevCore.Api/Protocols/DiscernStationRunner.cs`, `ProbeRefreshDiscernReweigh.cs`; `services/agent-service/app/services/sports_analyzer.py` (`discern` block, `watch_for`)
