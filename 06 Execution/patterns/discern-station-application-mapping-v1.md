---
title: "Discern Station Application Mapping v1"
type: "execution-pattern"
date: "2026-07-04"
status: "draft"
project: "DAI"
slice: "Discern Station Application Mapping v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - protocol
  - discern
  - cognitive-factory
  - architecture
  - calibration
related:
  - "02 Platform/architecture/cognitive-factory/protocol-node-specs.md"
  - "02 Platform/architecture/cognitive-factory/phases/discern.md"
  - "02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md"
  - "06 Execution/reconciliations/prompt-route-attribution-contract-v1.md"
  - "06 Execution/reconciliations/source-readiness-preflight-gate-v1.md"
  - "06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# Discern Station Application Mapping v1

## purpose

Map DAI's *current* sports decision pipeline into the Cognitive Protocol's **Discern** station and its three
micro-actions -- Weigh, Contrast, Stress -- and define what Discern reads, produces, preserves as residue, and
hands forward to Decide and Synthesize. This is a **design-only** mapping. It changes no runtime behavior, no
prompt text, no prompt selection, no confidence formula, no buyer output, and no metrics. It proposes a
`DiscernAssessment` read-model contract in docs only; nothing here is implemented.

## context

The canonical Discern node contracts already exist in `protocol-node-specs.md` (2026-05-18) and `phases/discern.md`
-- built on the **signal-quality surface** (`SignalQualityEvaluator`, `SignalFollowUpEvaluator`, sharp/public,
grounded-signal counts). Those contracts are authoritative and are **not** restated or changed here.

Since those specs were written, a second evidence layer landed through the v8 / registry / attribution / readiness
slices: **source-readiness preflight**, **observed vs selected data regime**, **prompt-path / route attribution**,
**registry-vs-live routing**, and the **reconciliation residue contract**. That layer is decision-relevant but is
not yet mapped into the Discern station. This doc is the bridge: it maps both the signal-quality layer (by
reference) and the newer route/regime/readiness/residue layer (in full) onto Weigh / Contrast / Stress, so a future
agent can reason about Discern over the *whole* current evidence surface.

## scope

**Included:** a mental model for Discern; what Discern consumes from Perceive and Interrogate; a current-pipeline
evidence field map; the Weigh / Contrast / Stress application mapping over that evidence; a proposed (docs-only)
`DiscernAssessment` contract; the residue Discern should preserve; what stays outside Discern; a short Decide /
Synthesize preview; and a minimum-viable future implementation path.

**Excluded:** any runtime/code/prompt/selection/confidence/buyer/metrics/schema change; a full Decide or Synthesize
design; enabling registry routing; implementing `DiscernAssessment`; restating or altering the signal-quality node
contracts in `protocol-node-specs.md` (referenced, not modified).

## discern mental model

Discern is the third station: **Perceive → Interrogate → Discern → Decide → Synthesize.** In one sentence, per
doctrine: *Discern decides what counts as evidence and how much weight each signal carries.* It runs **after**
collection and interrogation and **before** the decision.

- **Discern judges; it does not decide.** It grades, compares, and stress-tests evidence and hands **judgment
  conditions** to Decide. It never sets the lean, the posture, or the final confidence number.
- **Deterministic backbone, model narrative on top.** The platform owns the structured grades
  (`SignalQualityEvaluator`, `block_aggressive_posture`); the model contributes only the `weigh` / `contrast` /
  `stress` narrative, which must agree with the deterministic grades.
- **Two evidence layers, one station.** (1) signal-quality layer -- per-signal grounded/quality/decision-use (owned
  by the node specs). (2) route/regime/readiness/residue layer -- was this run's evidence *ready*, and was its
  prompt path *attributable* (new; mapped here). Discern should weigh, contrast, and stress both.
- **Timing honesty.** In today's runtime the 11 model-owned micro-actions (incl. Discern's `weigh`/`contrast`/
  `stress` and Decide's `resolve`/`position`/`justify`) are emitted by a **single analyze call**; `Confidence` /
  `ConfidenceBand` are computed deterministically by `SportsEvaluator` **after** that call. The station ordering is
  therefore **logical/doctrinal, not a sequence of separate calls.** Any future `DiscernAssessment` is an
  aggregation over already-produced fields, not a new model round-trip.

## what discern consumes (from perceive and interrogate)

Answering Q1. Discern reads, it does not retrieve:

- **From Perceive:** `perceive.detect / frame / aim` (what signals exist, the factual frame, what matters); plus the
  retrieval-stamped `GroundedSignals[]`, `MissingSignals[]`, `SignalAvailability[]` (the deterministic grades that
  are Weigh's backbone).
- **From Interrogate:** `interrogate.question` (strongest counter-case), `interrogate.probe` (deterministic
  missing-primary summary), `interrogate.verify` (alternate explanation tested against staged evidence).
- **From the pre-run readiness layer (if captured):** the `GET /source-readiness` classification
  (`starterReadiness`, `marketReadiness`, `predictedObservedDataRegime`) -- a **pre-model** screen that is read-only
  and not persisted on the run today. Discern references it when available; it does not depend on it.

Discern does **not** consume Decide/post-decision outputs (`lean`, `leanSide`, `posture`, final `Confidence`) as
inputs -- those are downstream. Treating them as Discern inputs would be circular.

## current pipeline evidence inventory (field map)

Per Phase 2. `pre/post` = relative to the analyze model call. `persist` = stored on the run/artifact today.
`buyer` = buyer-visible. `D-read` = Discern should read it. `D-write` = Discern writes it vs merely references it.

| field | producer | pre/post | persist | buyer | D-read | D-write/ref |
|---|---|---|---|---|---|---|
| source-readiness result | `GET /source-readiness` (preflight, reuses `SportsRetriever`) | pre | no (read-only, no run/snapshot write) | no | yes | reference |
| starterReadiness | source-readiness classifier | pre | no | no | yes | reference |
| marketReadiness (posted?, book count) | source-readiness classifier | pre | no | no | yes | reference |
| predictedObservedDataRegime | source-readiness classifier | pre | no | no | yes | reference |
| observedDataRegime | `observed_data_regime(inputs)` (migration_readiness) | at-run, deterministic | yes (provenance → /rows) | no | yes | reference |
| selectedDataRegime | registry selection | at-run | yes (null on live) | no | yes | reference |
| selectedPromptPath | route decision (`live/registry/fallback`) | at-run | yes | no | yes | reference |
| promptSource | route decision (`live/registry`) | at-run | yes | no | yes | reference |
| recipeId / version / assembledHash | registry selection | at-run | yes (null on live) | no | yes | reference |
| attributionStatus / attributionReason | route decision (`complete/partial/unattributed`) | at-run | yes | no | yes | reference |
| livePromptTemplateKey | constant marker | at-run | yes (null on registry) | no | yes | reference |
| market book count | market context / retrieval | pre | yes (context) | no | yes | reference |
| market consensus side | market context | pre | yes | no | yes | reference |
| market agreement | derived from market context | pre | partial | no | yes | reference |
| source depth | retrieval / `SignalAvailability` | pre | yes | no | yes | reference |
| SignalAvailability (Quality, DecisionUse, ConfidenceEffect incl. `block_aggressive_posture`) | `SignalQualityEvaluator` | pre (retrieval-side) | yes | no | yes | **Weigh backbone; writes `discern.weigh` narrative** |
| SignalFollowUps (status/reason/fallbackType/equivalence) | `SignalFollowUpEvaluator` | pre | yes | no | yes | reference |
| Confidence / ConfidenceBand | `SportsEvaluator` (deterministic) | post | yes | confidence: yes | yes | **reference only (never recompute)** |
| advertised strength | buyer projection | post | yes/projected | **yes** | yes | **reference only (never change)** |
| evidence sufficiency (band gate) | evidence-sufficiency gate | post | yes | influences buyer | yes | reference only |
| lean / leanSide | Decide.Resolve extraction | post (downstream) | yes (`AgentRun.LeanSide`) | lean: yes | no (downstream) | not an input |
| no-decision state | Decide.Resolve (`leanSide` null) | post (downstream) | yes | partial | no (downstream) | Discern emits *abstentionPressure* that feeds it |
| settlement outcome | `AgentRunOutcome` | post-game | yes | no | when available | reference |
| run evaluation (correct/incorrect/inconclusive) | `RunEvaluator` → `AgentRunEvaluation` | post-game | yes | no | when available | reference |
| pooled calibration (byRoute / byObservedRoute) | `pooled_calibration.py` | offline aggregate | computed | no | yes | reference |
| route-level historical performance | evaluations grouped by route | offline | computed | no | yes | reference |
| reconciliation residue (source, sourceRef, notes) | residue contract on settlement writes | post-game | yes (on outcome) | no | when available | reference |

## discern micro-action mapping

Per Phase 4. The signal-quality behavior of each micro-action is already specified in `protocol-node-specs.md` and
is **inherited unchanged**; below extends each to the route/regime/readiness/residue layer.

### Weigh

- **inputs:** `SignalAvailability[]` (Quality/DecisionUse/ConfidenceEffect), `GroundedSignals[]`,
  `MissingSignals[]`, source depth, market book count, market agreement, source-readiness (when captured),
  `observedDataRegime`, `attributionStatus`, `recipe*`/`assembledHash` presence, and -- when available for
  calibration -- route-level historical performance.
- **logic:** grade each evidence stream by strength, completeness, reliability, and freshness. The deterministic
  grades are authoritative; Weigh's model narrative must agree with them. Extend to: is the market *ready* (posted,
  enough books)? is the starter *ready*? is the run's prompt path *attributable* (`attributionStatus = complete`
  vs `partial` vs `unattributed`)? does `observedDataRegime` match a depth the evidence actually supports?
- **outputs:** the `discern.weigh` narrative (unchanged) plus, in the proposed contract, an `evidenceWeights[]`
  aggregation (see below). No number is invented.
- **examples:** starter readiness; market readiness / book count; source sufficiency and depth; identity safety;
  route-attribution completeness; settlement/calibration evidence when available.
- **residue:** per-signal strength + reason; readiness snapshot; attribution completeness -- so a reviewer can see
  *why* a signal was weighted as it was.
- **must not:** admit weak/stale evidence as strong; recompute or override `Confidence`/`ConfidenceBand`; let the
  model narrative override the deterministic grades; fabricate a book count or readiness it does not have.

### Contrast

- **inputs:** two or more grounded streams: `market` vs `sharp_public`; DAI's emerging read vs market consensus;
  `observedDataRegime` vs `selectedDataRegime`; `selectedPromptPath` (registry vs live); source-readiness prediction
  vs observed regime; home vs away lean support; confidence vs evidence sufficiency; current route vs route-level
  history.
- **logic:** classify each pair as aligned / conflicting / asymmetric / unknown, with a severity and a reason.
  Single-signal runs (e.g. MLB `starting_pitching`-only) have little to contrast -- say so rather than invent
  divergence.
- **outputs:** `discern.contrast` narrative (unchanged) plus a proposed `contrasts[]` of `ContrastFinding`.
- **examples:** DAI lean vs market consensus; starter edge vs market favorite; `observedDataRegime` vs
  `selectedDataRegime`; registry path vs live path; readiness vs observed regime; confidence vs evidence
  sufficiency; historical route performance vs current route.
- **residue:** each contrast pair, relationship, severity, reason -- the disagreements Decide most needs to see.
- **must not:** invent divergence; treat one signal as corroboration of itself; treat `observedDataRegime !=
  selectedDataRegime` as an error (on the live path `selectedDataRegime` is legitimately null).

### Stress

- **inputs:** `MissingSignals[]`, `perceive.*`, `discern.weigh`, `discern.contrast`, readiness gaps, attribution
  gaps (`partial`/`unattributed`), fallback/route state, regime mismatch, sample-size / calibration-gate state,
  and buyer-copy risk surfaces.
- **logic:** name the fragility, failure mode, and decision risk **before** Decide -- and whether each fragility
  *blocks* a confident decision. Operates on weighed evidence, which is why Stress's canonical home is Discern.
- **outputs:** `discern.stress` narrative (unchanged) plus a proposed `stressors[]` of `StressFinding`, each with a
  `blocksDecision` flag and recommended handling.
- **examples:** missing starter; missing / shallow market; identity ambiguity; route fallback; regime mismatch;
  market disagreement; thin evidence; stale source; no-decision pressure; sample-size insufficiency; calibration
  gate failure; buyer-copy risk.
- **residue:** each stressor, severity, whether it blocks, and recommended handling -- the audit trail for why a
  run was held, abstained, or flagged for review.
- **must not:** emit a vague fragility; fabricate an injury/form risk that is not grounded; make the final
  abstain/decide call (it prepares abstention *pressure*, Decide resolves it).

**Discern's boundary in one line:** it prepares the **judgment conditions** for Decide; it does not make the
decision.

## proposed DiscernAssessment contract (docs-only)

Per Phase 3. A read-model that **aggregates** existing per-node outputs and the evidence field map into one
inspectable object for Decide, audit, and calibration. **Not implemented; not runtime; proposed for a future,
separately-approved slice.** It recomputes nothing -- it references the deterministic `Confidence`/`ConfidenceBand`,
`block_aggressive_posture`, attribution, and readiness that already exist.

```
DiscernAssessment {
  agentRunId
  sourceProvider
  externalGameId
  observedDataRegime
  selectedDataRegime
  selectedPromptPath
  recipeId
  evidenceWeights: EvidenceWeight[]
  contrasts: ContrastFinding[]
  stressors: StressFinding[]
  confidenceCeiling      // ADVISORY only: a proposed pre-Decide ceiling. Does NOT alter the
                         //  deterministic SportsEvaluator confidence unless a future approved slice wires it.
  decisionPressure: DecisionPressure
  abstentionPressure     // pressure toward no-decision; Decide.Resolve owns the actual call
  needsHumanReview       // bool: attribution unattributed, regime mismatch, or calibration-gate failure
  discernSummary
  residueNotes
}

EvidenceWeight  { signal, value, strength, reason }
ContrastFinding { leftSignal, rightSignal, relationship: aligned|conflicting|asymmetric|unknown, severity, reason }
StressFinding   { stressor, severity, blocksDecision: bool, reason, recommendedHandling }
DecisionPressure{ leanSupport, abstentionSupport, uncertaintyDrivers, decisionReadiness }
```

**Consumed by:** Decide (primary). Also read by audit / `/dev/artifacts`-style inspection and by calibration
(route-level residue). It is a superset *view*; it does not replace `SignalAvailability[]`, `SignalFollowUps[]`, or
the canonical `CognitiveProtocol.Discern` block, which remain the persisted sources of truth.

**Safety rails baked into the contract:** `confidenceCeiling` is advisory (no confidence-formula change);
`abstentionPressure`/`decisionPressure` are inputs to Decide, not the decision; no buyer-visible or advertised-
strength field appears; nothing here is written back onto the buyer surface.

## residue discern should preserve (audit + calibration)

Answering Q6. Discern's durable residue = `evidenceWeights` (why each signal weighed as it did) + `contrasts`
(the disagreements) + `stressors` (the fragilities and what blocked) + the route/regime/attribution snapshot
(`observedDataRegime`, `selectedDataRegime`, `selectedPromptPath`, `attributionStatus`, `recipe*`). This residue is
what lets outcome reconciliation later ask *"was the read wrong because of a fragility Discern already named?"* and
ties naturally to the reconciliation residue contract (`source` + `sourceRef` + `notes` on settlement writes).

## what must remain outside discern

Answering Q7. Outside Discern: **retrieval / source fetching** (platform plumbing, pre-Perceive); **the confidence
number and band** (`SportsEvaluator`, deterministic); **the lean/direction** (Decide.Resolve); **the posture enum**
(Decide.Position); **the calibration rationale** (Decide.Justify); **buyer-visible output and advertised strength**
(Synthesize / buyer projection); **prompt selection and registry routing** (route decision layer, default-off);
**metrics denominator** (unchanged). Discern may *reference* these; it owns none of them.

## decide / synthesize preview (future work)

Per Phase 5 -- **marked future work; not designed here.** Decide consumes `DiscernAssessment`:

- **Resolve** -- choose decision vs abstention vs needs-review, using `decisionPressure` / `abstentionPressure` /
  `needsHumanReview`.
- **Position** -- set lean/posture (clamped enum) if warranted, respecting `block_aggressive_posture`.
- **Justify** -- evidence-backed rationale; the confidence number stays deterministic.

Synthesize then consumes Decide output: **Integrate** (combine decision + evidence + route + caveats),
**Compose** (internal + buyer-safe structures), **Deliver** (the appropriate output surface). Full Decide/Synthesize
mapping is the **next** protocol-mapping slice.

## minimum viable future implementation path

Answering Q8, lowest-risk first, each a separately-approved slice (mirrors the runtime-activation ladder --
observe before execute; nothing here is dormant-wired into production):

1. **Docs/contract (this slice).** Freeze the mapping + `DiscernAssessment` shape. No code.
2. **Read-only projection.** Build `DiscernAssessment` as a *derived view* over already-persisted fields (like
   `ProtocolView` / `byObservedRoute`) -- no new model call, no new persisted column, surfaced on the artifact
   inspection endpoint only. Purely additive observability.
3. **Calibration use.** Let offline calibration read the residue (stressors/contrasts vs outcomes) to sharpen route
   and confidence-bucket analysis -- read-only, no formula change.
4. **(Gated, later)** Only if evidence supports it and separately approved: let advisory `confidenceCeiling` /
   `abstentionPressure` inform Decide. This touches decision/confidence behavior and is **out of scope** until its
   own Evidence-Readiness-gated slice.

## safety / non-actions

Docs-only. **Not changed:** runtime code; prompt text; prompt-selection behavior; registry routing (stays
default-off); confidence formula; advertised strength; evidence sufficiency; buyer output; metrics denominator; DB
schema. **Not done:** paid model calls (0); sports runs (0); reconciliation writes (0); external sports API calls
(0); DB migrations/backfills (0); no `DiscernAssessment` implemented; no dormant runner wired into production. The
canonical Discern node contracts in `protocol-node-specs.md` are referenced and left unchanged. v8 generation
budget remains closed at 10.

## next recommended slice

If the pending v8 cohort games are **Final** on StatsAPI: **Settle-and-Reassess Registry-Routed v8 Cohort** (the
standing runtime priority) takes precedence. If they are **not** Final: **Decide Station Application Mapping v1** --
apply the same treatment to Decide (Resolve / Position / Justify) consuming this `DiscernAssessment`, continuing the
protocol-mapping thread while settlement waits on finality.

## related

- [[protocol-node-specs]] -- authoritative per-node Discern contracts (signal-quality layer), unchanged here.
- `02 Platform/architecture/cognitive-factory/phases/discern.md` -- Discern responsibility doctrine.
- [[prompt-route-attribution-contract-v1]] -- observed/selected regime + prompt-path attribution fields.
- [[source-readiness-preflight-gate-v1]] -- the pre-model readiness screen.
- [[canonical-reconciliation-residue-contract-v1]] -- settlement residue Discern's audit trail ties into.
