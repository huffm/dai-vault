---
title: "Discern Micro-Protocol Design v1"
type: "execution-pattern"
date: "2026-07-04"
status: "draft"
project: "DAI"
slice: "Discern Micro-Protocol Design v1"
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
  - "06 Execution/patterns/discern-station-application-mapping-v1.md"
  - "02 Platform/architecture/cognitive-factory/protocol-node-specs.md"
  - "02 Platform/architecture/cognitive-factory/phases/discern.md"
  - "02 Platform/architecture/cognitive-factory/phases/interrogate.md"
  - "02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md"
  - "06 Execution/reconciliations/prompt-route-attribution-contract-v1.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# Discern Micro-Protocol Design v1

## purpose

Mature the Discern station from an application mapping ([[discern-station-application-mapping-v1]]) into a concrete
**micro-protocol** with operational semantics comparable to Interrogate's Question -> Probe -> Verify loop. Define,
for Weigh -> Contrast -> Stress: the deterministic rules, the allowed LLM-assisted roles, the input/output
contracts, the residue, and how Discern prepares Decide **without making the final decision**. Design-only: no
runtime, prompt, selection, confidence, buyer, metrics, or schema change.

## context

Interrogate matured because it split a narrative act into a **typed, deterministic backbone with a subordinate
model narrative**: Question/Verify are model-emitted, but Probe is fully deterministic
(`CognitiveProtocolBuilder.BuildProbe` over `SignalFollowUpRecord[]`), and the model "names candidates" while "the
platform grades them" (`phases/interrogate.md`). Discern today is uneven by comparison: Weigh has a deterministic
backbone (`SignalQualityEvaluator`) but Contrast and Stress are model-narrative-only (`protocol-node-specs.md`,
legacy `discern.listen` / `discern.test`). This slice designs the missing typed backbone for all three, extending
the mapping v1 evidence field-map, so Discern can become as auditable and resumable as Interrogate.

Nothing here changes the canonical node contracts in `protocol-node-specs.md`; it proposes the target
micro-protocol they would grow into, and refines the docs-only `DiscernAssessment` contract to v2.

## scope

**Included:** Interrogate lessons distilled for Discern; Weigh/Contrast/Stress operational semantics (deterministic
rules + LLM role + I/O + residue + forbidden); a deterministic rule taxonomy; the LLM role boundary and grounding
requirements; `DiscernAssessment` v2; the Decide handoff contract; the smallest future implementation path.

**Excluded:** any runtime/code/prompt/selection/confidence/buyer/metrics/schema change; implementing
`DiscernAssessment` or any Discern runner; a full Decide or Synthesize design; enabling registry routing; altering
the node specs (referenced, not changed).

## interrogate lessons for discern

What made Question -> Probe -> Verify operationally strong, and what Discern copies structurally:

1. **A typed deterministic backbone under the narrative.** Probe is derived from typed fields
   (`Reason=="primary_signal_missing"`, `DecisionUse=="missing_confirmation"`, `FallbackType=="lateral_proxy"` ->
   parent), not from prose. **Discern copy:** each micro-action gets a typed finding produced by deterministic
   rules; the model narrative sits on top and must agree.
2. **Fail-closed whitelist.** Only signals with a doctrinal template emit a Probe sentence; uncharacterized signals
   are silently dropped so Probe "never fabricates." **Discern copy:** an unsupported finding is dropped or flagged
   `unsupportedClaims`, never promoted into support.
3. **Determinism = dedup + canonical ordering.** `MissingProbeSignals` uses an ordinal `HashSet` sorted by canonical
   name, so identical inputs always yield identical output -> resumable and auditable. **Discern copy:** all typed
   findings are dedup'd and canonically ordered.
4. **Single-sourced structured + narrative.** `MissingProbeSignals` single-sources both the string probe and the
   structured `ProbeRequest` so "the two forms can never diverge." **Discern copy:** the `DiscernAssessment`
   structured findings and the `discern.*` narrative derive from one deterministic finding set.
5. **Descriptive-only advisory layer.** `MapConfidenceEffect` is explicitly "descriptive only -- ProbeRequest
   changes no confidence rule." **Discern copy:** advisory outputs (e.g. `confidenceCeilingAdvisory`) never mutate a
   gate, weight, or the confidence formula.
6. **Model proposes/explains; platform grades/blocks.** "Interrogate names candidates. Discern grades them."
   **Discern copy:** the model may surface tensions and rationale; deterministic rules own weights, blockers, and
   gates.
7. **Guardrails police the model and fail safe to null.** Forbidden phrases + `SportsQualityChecker` +
   null-on-malformed. **Discern copy:** the same guardrail posture applies to Discern narrative.

## discern micro-protocol

Expected posture (per the working model): **Weigh deterministic-heavy; Contrast hybrid but typed and grounded;
Stress deterministic-first, scenario/blocker based.** LLM output may explain or surface tensions but must never
silently alter gates, weights, or blockers.

### Weigh (deterministic-heavy)

- **purpose:** determine how much each evidence class should matter -- strength, reliability, freshness,
  completeness, importance.
- **inputs:** `SignalAvailability[]` (Quality/DecisionUse/ConfidenceEffect), `GroundedSignals[]`/`MissingSignals[]`,
  source depth, market book count, source-readiness (when captured), `observedDataRegime`, `attributionStatus`,
  `recipe*`/`assembledHash` presence.
- **deterministic rules:** source-completeness, source-reliability, source-freshness (where captured), market-depth,
  regime-attribution completeness (see taxonomy). These produce a typed `EvidenceWeight` per class. The existing
  `SignalQualityEvaluator` grades are the authoritative backbone.
- **optional LLM role:** a one-line `weigh` narrative that *explains* the weights in plain language; it must agree
  with the deterministic grades and cite the fields it rests on.
- **outputs:** `evidenceWeights[]` (typed) + the `discern.weigh` narrative (subordinate).
- **residue:** per-class strength + reason + the readiness/attribution snapshot behind it.
- **forbidden:** recompute or override `Confidence`/`ConfidenceBand`; admit weak/stale as strong; let the narrative
  override the deterministic grades; invent a book count or freshness it does not have.

### Contrast (hybrid, typed and grounded)

- **purpose:** determine whether evidence classes align, conflict, are asymmetric, or are unknown relative to each
  other.
- **inputs:** two or more graded classes: market vs sharp/public; DAI emerging read vs market consensus;
  `observedDataRegime` vs `selectedDataRegime`; registry vs live path; readiness-prediction vs observed regime;
  confidence vs evidence sufficiency; route-history vs current route.
- **deterministic rules:** typed pairwise checks (market-agreement, observed-vs-selected, confidence-vs-sufficiency
  -- see taxonomy). Each emits a `ContrastFinding{relationship: aligned|conflicting|asymmetric|unknown, severity,
  reason}` over a **named pair of typed fields**. Single-signal runs (MLB starter-only) legitimately have nothing to
  contrast -> emit `unknown`/empty, never invented divergence.
- **optional LLM role:** surface *candidate* tensions the static pairwise checks did not enumerate, as advisory
  findings that must map to real fields; unmapped candidates go to `unsupportedClaims`, not into the contrast set.
- **outputs:** `contrastFindings[]` (typed) + `discern.contrast` narrative.
- **residue:** each pair, relationship, severity, reason.
- **forbidden:** invent divergence; treat a live-path null `selectedDataRegime` as a mismatch/error; treat a signal
  as corroboration of itself; let an LLM-surfaced tension enter the typed set without field grounding.

### Stress (deterministic-first, scenario/blocker based)

- **purpose:** determine what could break, weaken, or block the decision posture, and whether each fragility
  **blocks** a confident decision.
- **inputs:** `MissingSignals[]`, the Weigh/Contrast findings, readiness gaps, attribution gaps, fallback/route
  state, regime mismatch, sample-size/calibration state, identity-safety state, buyer-copy-risk surfaces, and the
  unresolved Interrogate questions.
- **deterministic rules:** identity-safety, route-fallback, sample-size/calibration-gate, buyer-copy-risk,
  abstention-pressure, needs-human-review (see taxonomy). Each emits a `StressFinding{stressor, severity,
  blocksDecision, reason, recommendedHandling}`. `blocksDecision` is set by a **deterministic** rule (e.g. ambiguous
  identity blocks), never by the model.
- **optional LLM role:** propose *stress questions* / scenario narratives that make a fragility legible; advisory
  only, and each must reference a typed finding.
- **outputs:** `stressFindings[]` (typed) + `discern.stress` narrative.
- **residue:** each stressor, severity, `blocksDecision`, recommended handling.
- **forbidden:** emit a vague fragility; fabricate an injury/form risk; set or clear a blocker from the narrative;
  make the abstain/decide call (Stress prepares abstention *pressure*; Decide.Resolve owns the call).

## deterministic rule taxonomy

Per Phase 3. `blocks` = whether the rule can set `blocksDecision`. "Now" = implementable from fields persisted
today; "future" = needs a new capture (flagged honestly).

| category | signal read | rule type | possible output | blocks | feeds | now/future |
|---|---|---|---|---|---|---|
| source completeness | `GroundedSignals`/`MissingSignals`/`SignalAvailability.Status` | presence/threshold | complete/partial/absent | no | Weigh | now |
| source freshness | source/readiness timestamps | staleness threshold | fresh/stale/unknown | no | Weigh/Stress | **future** (freshness largely not captured today) |
| source reliability | `SignalAvailability.Quality`/`Source` | quality grade | strong/usable/unavailable | no | Weigh | now |
| identity safety | resolved team/game identity, `externalGameId` | match/ambiguity | safe/ambiguous | **yes** | Stress | now |
| market depth | market book count | min-book threshold | deep/shallow/absent | no | Weigh/Stress | now |
| market agreement | market consensus vs sharp/public vs emerging read | pairwise agreement | aligned/conflict/asymmetric | no | Contrast | now |
| regime attribution | `observedDataRegime`/`attributionStatus`/`recipe*` | completeness | complete/partial/unattributed | no | Weigh/Stress | now |
| route fallback | `selectedPromptPath`/`fallbackReason` | fallback detection | none/fallback | no | Stress | now |
| observed-vs-selected regime | `observedDataRegime` vs `selectedDataRegime` | null-aware match | match/mismatch/na-live | no | Contrast/Stress | now |
| confidence-vs-evidence sufficiency | `Confidence`/`ConfidenceBand` vs `EvidenceRichness`/sufficiency band | threshold cross-check (reference only) | consistent/over-confident-flag | no | Contrast/Stress | now |
| sample-size/calibration gate | pooled bucket counts / route-level n | min-sample gate | sufficient/thin/gate-fail | conclusions only | Stress | now |
| buyer-copy risk | advertised strength / lean strength vs evidence | overclaim detection | safe/risk | no | Stress | now |
| abstention pressure | aggregate of missing primaries + blockers + thin evidence | aggregate threshold | low/med/high | no | Decide (DecisionPressure) | now |
| needs-human-review | any-of: unattributed, regime mismatch, calibration-gate fail, identity ambiguous | any-of trigger | true/false | no | Decide | now |

Note: `confidence-vs-evidence sufficiency` **reads** the deterministic confidence and only flags inconsistency; it
never recomputes it (that would touch the confidence formula, which is out of scope).

## LLM role boundary

**Allowed (advisory, grounded):** summarize weighted evidence in natural language; surface candidate tensions the
static pairwise checks missed; propose stress questions/scenarios; produce human-readable rationale fragments.

**Forbidden (discretionary):** invent evidence; override source readiness, identity safety, or route provenance;
change the confidence formula; set lean/posture; promote or enable registry routing; bypass a deterministic
blocker; convert missing evidence into support.

**Grounding requirements for any LLM-assisted output:** it must (1) cite the source field(s) it rests on; (2) map to
a typed finding (`EvidenceWeight` / `ContrastFinding` / `StressFinding`); (3) **fail closed** -- an ungroundable
claim is routed to `unsupportedClaims[]`, not into the finding set; (4) remain advisory unless a deterministic rule
independently confirms it. The model may explain and surface; it may never silently alter a gate, weight, or
blocker.

## DiscernAssessment v2

Per Phase 5. Refines the mapping-v1 contract. **Docs-only; not implemented; not runtime.** It aggregates existing
per-node outputs + the typed findings; it recomputes nothing. `det` = deterministic, `llm` = LLM-assisted
(advisory, grounded per above).

| field | type | producer | consumer | det/llm | persisted | buyer-visible |
|---|---|---|---|---|---|---|
| evidenceWeights[] | `EvidenceWeight[]` | Weigh rules | Decide.Justify | det (narrative llm) | future | no |
| contrastFindings[] | `ContrastFinding[]` | Contrast rules | Decide.Justify | det + llm-surfaced (grounded) | future | no |
| stressFindings[] | `StressFinding[]` | Stress rules | Decide.Resolve/Justify | det (`blocksDecision` det; narrative llm) | future | no |
| decisionPressure | `{leanSupport, abstentionSupport, uncertaintyDrivers, decisionReadiness}` | aggregate rules | Decide.Position | det | future | no |
| abstentionPressure | enum low/med/high | abstention-pressure rule | Decide.Resolve | det | future | no |
| needsHumanReview | bool | needs-human-review rule | Decide.Resolve | det | future | no |
| confidenceCeilingAdvisory | number? | confidence-vs-sufficiency rule | Decide.Justify | det, **advisory only** | future | no |
| unsupportedClaims[] | string[] | LLM grounding filter | audit/calibration | llm-flagged, det-filtered | future | no |
| unresolvedQuestions[] | string[] | Interrogate Question minus Verify | Decide.Resolve / Stress | det | future | no |
| verifiedInterrogateAnswers[] | string[] | Interrogate Probe+Verify | Weigh (enriched evidence) | det | future | no |
| discernSummary | string | Weigh/Contrast/Stress narrative | audit / Decide | llm (grounded) | no | no |
| residueNotes | string | all rules | audit/calibration | det | future | no |

`confidenceCeilingAdvisory` renames v1's `confidenceCeiling` to make the non-binding nature explicit. `unsupported
Claims` / `unresolvedQuestions` / `verifiedInterrogateAnswers` are new in v2 and encode the Interrogate handoff
(Q7). The contract is a superset **view**; it does not replace `SignalAvailability[]`, `SignalFollowUps[]`, or the
canonical `CognitiveProtocol.Discern` block.

## what discern consumes from interrogate (Q7)

- **Question -> unresolvedQuestions[]:** open questions/counter-cases that Probe+Verify did NOT satisfy become
  Discern stressors and feed abstention pressure.
- **Probe -> weigh inputs:** the deterministic missing-primary gap set (`MissingProbeSignals`) tells Weigh which
  primaries are absent and Stress which gaps remain.
- **Verify -> verifiedInterrogateAnswers[]:** questions the probe answered enough to enrich the artifact become
  admitted evidence Weigh can grade. Per doctrine, "Interrogate names candidates; Discern grades them" -- Discern
  applies the Sharp/Public Fallback Ladder classification, Interrogate does not.

## handoff to decide

Per Phase 6. **Decide receives** (via `DiscernAssessment`): the weighted evidence surface (`evidenceWeights`), the
conflict/tension map (`contrastFindings`), the blockers/stressors (`stressFindings` with `blocksDecision`),
`abstentionPressure`, `needsHumanReview`, `confidenceCeilingAdvisory`, and `residueNotes`.

**Decide must not receive:** raw unverified LLM speculation; unsupported stress narratives (they stay in
`unsupportedClaims`, out of the finding set); hidden weights (every weight is a typed `EvidenceWeight` with a
reason); untraceable confidence changes (the ceiling is advisory and the confidence number stays deterministic).

**How Decide's micro-actions use Discern (full Decide design deferred):**

- **Resolve** consumes blockers (`stressFindings.blocksDecision`), `needsHumanReview`, and `abstentionPressure` to
  choose decision vs abstention vs needs-review.
- **Position** consumes `decisionPressure` (leanSupport/abstentionSupport) to set lean/posture if warranted,
  honoring `block_aggressive_posture`.
- **Justify** consumes `evidenceWeights`, `contrastFindings`, and `stressFindings` for the rationale; the confidence
  number stays deterministic (`SportsEvaluator`).

## smallest future implementation path (Q10)

Lowest-risk first, each a separately-approved slice; nothing dormant-wired into production:

1. **This slice: docs/contract.** Freeze the micro-protocol semantics + rule taxonomy + `DiscernAssessment` v2. No
   code.
2. **Read-only typed findings.** Implement the "now" deterministic rules as a derived, read-only projection over
   already-persisted fields (mirroring `BuildProbeRequest`/`byObservedRoute`) -- structured `EvidenceWeight` /
   `ContrastFinding` / `StressFinding`, surfaced on the artifact inspection endpoint only. Additive; no model call,
   no new persisted column, no gate.
3. **Single-source the narrative.** Have the `discern.*` narrative and the structured findings derive from one
   source (the Interrogate Probe pattern) so they cannot diverge.
4. **Calibration use.** Let offline calibration read the residue (stressors/contrasts vs outcomes). Read-only, no
   formula change.
5. **(Gated, later)** Only if evidence supports it and separately approved: let advisory
   `confidenceCeilingAdvisory` / `abstentionPressure` inform Decide. Touches decision/confidence behavior -> its own
   Evidence-Readiness-gated slice, out of scope here.

## safety / non-actions

Docs-only. **Not changed:** runtime code; prompt text; prompt-selection behavior; registry routing (stays
default-off); confidence formula; advertised strength; buyer output; metrics denominator; DB schema. **Not done:**
paid model calls (0); sports runs (0); reconciliation writes (0); external sports API calls (0); DB
migrations/backfills (0); no `DiscernAssessment` or Discern runner implemented; no dormant runner wired into
production. The canonical node contracts in `protocol-node-specs.md` are referenced and unchanged. v8 generation
budget remains closed at 10.

## next recommended slice

If the pending v8 cohort games are **Final** on StatsAPI: **Settle-and-Reassess Registry-Routed v8 Cohort** (the
standing runtime priority) takes precedence -- settle each via the reconciliation residue contract. If they are
**not** Final: **Decide Station Application Mapping v1** -- apply the same treatment to Decide (Resolve / Position /
Justify) consuming this `DiscernAssessment` v2, continuing the protocol-design thread while settlement waits on
finality.

## related

- [[discern-station-application-mapping-v1]] -- the v1 mapping this design matures.
- [[protocol-node-specs]] -- authoritative per-node Discern/Interrogate contracts, unchanged here.
- `02 Platform/architecture/cognitive-factory/phases/interrogate.md` -- the maturity model copied structurally.
- `02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md` -- the ladder Discern applies to Probe candidates.
- [[prompt-route-attribution-contract-v1]] -- the route/regime attribution fields Weigh/Contrast/Stress read.
