# Confidence Calibration Rules v1

**date:** 2026-06-09
**status:** doctrine / spec only. no code, no `SportsEvaluator` change, no confidence constants changed, no buyer UI/projection change, no prompt change, no schema change, no source/probe-refresh change, no cost-guardrail change, no pricing change. implementation is a later slice, gated as described below.
**type:** confidence doctrine for the sports decision artifact factory. this document is the decision; it is kept embedded here rather than as a separate decisions/ stub (the existing `04 Products/sports-v1/decisions/` folder is a UI-design-decision convention, and this doctrine is large enough to own its own spec file).

## Purpose

Fresh Buyer Artifact Validation v1 found `confidence_high_for_partial_evidence` on 4 of 4 fresh MLB runs: confidence 0.75 ("high") on a single grounded signal (`starting_pitching`, `evidenceRichness = 1`). This slice defines how confidence should *behave* in the factory so a thin read is not advertised as a strong buyer read. It does not patch a number. It defines a doctrine that a future slice can implement safely, and that future outcome data can recalibrate without losing buyer honesty.

Primary question this doctrine answers: how should the system relate "how coherent the read is" to "how much grounding supports it" to "how strongly it tells a buyer," without pretending to a precision it has not yet earned?

## Scope

- define a two-axis confidence doctrine (confidence + evidence sufficiency) and how buyer-advertised strength derives from both.
- frame confidence as a system behavior (Thinking in Systems lens), not a scalar to tune.
- name the specific failure mode behind the MLB finding.
- recommend a later implementation direction (band-gating before numeric rewriting) without implementing it.

## Non-goals (this slice)

- no code; no change to `SportsEvaluator.cs` or its calibration constants.
- no buyer UI / projection change; no prompt change; no schema change.
- no source addition; no probe-refresh activation; no cost-guardrail or pricing change.
- no change to confidence, posture, lean, or decision values produced today.
- no push.

## Current behavior (what exists today)

The evaluate stage (`platform/dotnet/DevCore.Api/AgentRuns/SportsEvaluator.cs`) computes a calibrated confidence deterministically after the single analyzer model call. Its model is **relative to each competition's maximum grounded signal set**:

- `maxGrounded` = `CompetitionCatalog` max grounded signals for the competition.
- `groundedCount` = `min(retrieved grounded signals, maxGrounded)`.
- calibration:
  - `groundedCount == 0` (priors-only): dampening 0.75, clamp [0.30, 0.60].
  - `groundedCount < maxGrounded` (partial): dampening 0.90, clamp [0.35, 0.75].
  - `groundedCount == maxGrounded` (full): dampening 1.00, clamp [0.35, 0.85].
- band: `>= 0.70` high, `>= 0.50` medium, else low.

The code comments are explicit that the dampening factors and clamps are "documented interim estimates, not magic numbers ... they will be calibrated against actual outcomes once the learning loop exists." This doctrine is consistent with that statement and does not replace those constants.

Observed consequence (fresh data, 2026-06-09):
- MLB `maxGrounded == 1` (only `starting_pitching` is a grounded source today). A single grounded signal is therefore `groundedCount == maxGrounded` -> treated as **fully grounded** -> dampening 1.00, clamp up to 0.85. Analyzer 0.75 -> **0.75 "high"** on `evidenceRichness = 1`.
- NBA `maxGrounded == 3` (market, rest_schedule, sharp_public). Two grounded, sharp_public missing -> **partial** -> 0.75 * 0.90 = 0.675 on `evidenceRichness = 2`.

## The failure mode (named)

**Relative completeness masking absolute scarcity -- a system boundary problem.**

The evaluator measures completeness *inside the competition/source boundary*: "did we get all the grounded signals this competition can currently supply?" For MLB today that boundary is a single signal, so "1 of 1" scores as complete and is allowed to pass confidence through at up to 0.85.

The buyer, however, experiences confidence *across the whole artifact*, with no knowledge of the competition's source ceiling. To the buyer, "1 grounded signal" is thin evidence regardless of whether it is all the system can currently fetch. The evaluator optimizes a local variable (relative completeness) while ignoring a system variable (absolute evidence scarcity), and the result is a misleading output: a thin read that looks high-confidence.

This is the root of the `confidence_high_for_partial_evidence` finding. It is a boundary mismatch, not a wrong constant; lowering a number would treat the symptom while leaving the boundary error in place.

## System framing (Thinking in Systems)

Confidence is a behavior of the factory, not a field. Modeled as a system:

- **Flows (inputs):** grounded signals, missing signals, fallback/proxy signals, source availability, and the model's interpretation of them.
- **Stock (state):** accumulated **evidence sufficiency** for a read -- how much independent grounding has actually been gathered.
- **Derived state:** **confidence** -- how coherent/strong the read is *from the available evidence*.
- **Output:** **buyer-advertised strength** -- band/posture the system presents to a buyer.
- **Feedback loop:** **outcome reconciliation** -- reconciled game-day outcomes (`AgentRunOutcome -> AgentRunEvaluation`) that, over time, can recalibrate the confidence number.
- **Delay:** outcomes arrive well after artifact generation, so the feedback loop is slow; calibration cannot be closed at generation time.
- **Constraints:** source availability, model-call budget/cost (one model call per run; see Artifact Cost Guardrails v1 candidate).
- **Protected long-term stock:** **buyer trust.** It accumulates slowly and drains fast. A single thin read advertised as strong withdraws from it.

**Core systems insight:** a local optimization of numeric confidence can damage the larger system if it causes thin-evidence artifacts to be advertised as strong buyer reads. Confidence must therefore not be treated as a standalone scalar. It must be interpreted together with evidence sufficiency, and recalibrated later through outcome feedback. Optimizing the number alone, before the feedback loop exists, risks protecting a local metric while spending the protected stock (trust).

## The two-axis doctrine

Three distinct questions, three distinct answers. Keeping them separate is the doctrine.

1. **Confidence** -- "How coherent/strong is the read from the available evidence?"
   - The numeric, *relative* value produced today by `SportsEvaluator` (preserved unchanged by this doctrine).
   - It is allowed to be high when the read is internally coherent and the available grounding (however much there is) points one way.

2. **Evidence sufficiency** -- "How much independent grounding supports the read?"
   - An **absolute** measure derived from the artifact's existing `evidenceRichness` (absolute count of grounded signals), independent of the competition's source ceiling.
   - Expressed as a tier: **thin / moderate / rich**. Indicative provisional thresholds (not outcome-calibrated, to be revisited): thin = at most 1 grounded signal; moderate = 2; rich = 3 or more. These thresholds are deliberately simple and absolute; they are not tuned constants pretending to be predictive.

3. **Buyer-advertised strength** -- "How strongly should the system present this read to a buyer?"
   - **Derived from both** confidence and evidence sufficiency.
   - Hard rule: a run may not advertise "high" strength while evidence sufficiency is **thin**, regardless of the numeric confidence. Thin-evidence reads are surfaced at a capped strength (at most the middle band) with the reason recorded.
   - The suppression is recorded as an explicit **evidence humility constraint** (a named reason, e.g. "advertised strength capped: thin evidence -- 1 grounded signal"), so the cap is inspectable and auditable, not silent.

4. **Outcome reconciliation** -- "Was the confidence calibration actually predictive over time?"
   - The future feedback loop. It can move the *confidence number* once reconciled outcomes exist. It does not need to exist for the evidence-sufficiency humility cap to be correct.

### Why two axes (and not a single capped number)

Separating the axes is what keeps the system honest across the slow feedback loop:

- The **evidence-sufficiency gate** is a *buyer-honesty constraint*. It is true on first principles -- one signal is thin -- and needs no outcome data to justify. It can be implemented before reconciliation exists.
- The **numeric confidence** is a *provisional estimate*. It is the thing outcome reconciliation will recalibrate. It should be left as a relative value so future outcome data can move it cleanly.

If instead we collapse both into one capped number now, we (a) bake a new hand-tuned constant into the value the outcome loop must later untangle, and (b) lose the explicit humility reason the buyer surface and audit can show. The two-axis model lets outcome data recalibrate confidence later **without** removing the buyer-honesty cap, and without us having pretended the cap was an outcome-calibrated precision claim.

## Feedback-loop honesty (provisionality)

Until outcome reconciliation exists, all confidence numbers in the factory are **provisional**. This doctrine deliberately:

- avoids introducing new hand-tuned numeric constants that would masquerade as outcome-calibrated;
- keeps the existing interim `SportsEvaluator` constants in place rather than re-guessing them;
- locates the only new *first-principles* rule (thin evidence cannot read as high strength) on the absolute evidence axis, where it is defensible without outcome data.

This preserves entry 12's standing position (no confidence/posture *threshold* moves without reconciled-outcome evidence) while making explicit that an evidence-sufficiency humility cap is a different kind of change -- a presentation/honesty constraint, not a predictive threshold move.

## Recommended later implementation direction (NOT this slice)

When a future slice implements this doctrine, prefer **band-gating before numeric rewriting**:

1. **Preserve** the numeric confidence exactly as `SportsEvaluator` produces it today (no constant change), so the future outcome loop has a clean, unmodified signal to recalibrate.
2. **Derive** an evidence-sufficiency tier (thin/moderate/rich) from the existing absolute `evidenceRichness` -- no new retrieval, no new source.
3. **Gate** the buyer-advertised strength so a thin-evidence run cannot surface as "high," even when numeric confidence is >= 0.70.
4. **Record** the suppression as an evidence humility constraint with a human-readable reason, carried where the artifact already carries diagnostic/availability metadata, so it is inspectable and testable.

Explicitly deferred from that future slice as well (must not ride along): changing `SportsEvaluator` constants, moving the 0.70/0.50 numeric band thresholds, buyer UI redesign, prompt changes, schema/contract changes, source expansion, probe-refresh activation, posture/lean changes.

Sequencing note: numeric confidence *recalibration* should follow Outcome Reconciliation v1 (entry 12). The evidence-sufficiency humility gate does not depend on it and could land earlier in its own narrowly scoped slice. Artifact Cost Guardrails v1 remains a sensible precursor to larger calibration batches but is independent of this doctrine.

## Relationship to the deferred runtime decisions ledger

- **Entry 12 (Calibration proof before posture-threshold changes):** unchanged in substance -- no confidence/posture *threshold* moves without reconciled-outcome evidence. This doctrine refines its boundary: an evidence-sufficiency humility cap on *advertised strength* is a buyer-honesty presentation constraint, distinct from moving a confidence/posture threshold, and is the one part of this doctrine not gated on outcome data. Numeric confidence recalibration remains gated by entry 12. A one-line clarification was added to entry 12 pointing here.
- **Entry 4 (Confidence / posture / lean mutation in probe-refresh):** unchanged. This doctrine does not let any probe-refresh path mutate confidence/posture/lean. Entry 4's revisit trigger ("calibration evidence per entry 12 plus an explicit doctrine change") is still not satisfied: this is a doctrine about base evaluate-stage calibration and buyer presentation, not authorization for probe-refresh mutation, and the calibration-evidence half remains unmet. Entry 4's status is not changed.

## Acceptance / success criteria for this doctrine slice

- the two-axis model (confidence, evidence sufficiency, derived buyer-advertised strength, outcome reconciliation) is defined.
- the systems framing (flows, stock, derived state, output, feedback loop, delay, constraints, protected stock = buyer trust) is recorded.
- the MLB relative-completeness-vs-absolute-scarcity failure mode is documented against the observed 4/4 finding.
- the later implementation direction (band-gating before numeric rewriting, with a recorded humility constraint) is recommended and explicitly not implemented.
- no code or product semantics changed.

## What was not changed

- no `SportsEvaluator` code or constants
- no confidence/posture/lean/decision values
- no buyer UI / projection
- no FastAPI prompt or parser
- no schema or contract
- no source / probe-refresh / cost-guardrail / pricing change
- no `OPENAI_API_KEY` or secret change
