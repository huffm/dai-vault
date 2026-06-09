# Evidence-Sufficiency Band Gate v1

**date:** 2026-06-09
**status:** implemented. narrow Angular buyer-projection change + tests in `dai`; report/handoff/ledger in `dai-vault`. no prompt, model-call, cost-guardrail, source, probe-refresh, lean, raw-confidence, posture, schema, .NET, or FastAPI change.
**scope:** first runtime expression of the two-axis confidence doctrine -- derive evidence sufficiency and prevent thin-evidence artifacts from surfacing as high buyer-advertised strength.

## Purpose

Fresh Buyer Artifact Validation v1 found `confidence_high_for_partial_evidence` on 4/4 MLB runs (confidence 0.75, one grounded signal, evidenceRichness 1). Confidence Calibration Rules v1 specced the doctrine: confidence, evidence sufficiency, and buyer-advertised strength are separate; thin evidence should not advertise high strength. This slice implements that humility constraint at the deterministic projection layer while preserving the lean and the raw numeric confidence.

Primary question: can we preserve internal confidence and lean while adding a buyer-advertised strength gate that keeps thin-evidence reads humble? Answer: yes -- as a frontend projection gate, with zero backend/contract change.

## Scope

- derive an absolute evidence-sufficiency tier from the existing `evidenceRichness`.
- gate the buyer-advertised strength (the buyer confidence band) so thin evidence cannot show "High".
- record an internal humility reason when the cap fires.
- preserve lean, raw numeric confidence, posture, artifact contract, and cost telemetry.

## Non-goals

- no prompt, model-call, or cost-guardrail change.
- no source retrieval / probe-refresh change.
- no change to lean direction, raw confidence value, or confidence constants.
- no SportsEvaluator / .NET / FastAPI / schema / artifact-contract change.
- no buyer copy change beyond the band value itself (High -> Medium is plain language).
- no pricing/Stripe/auth/tenant/dashboard/deployment change. no Jera change. no push.

## Doctrine applied

Two-axis model from Confidence Calibration Rules v1:
- confidence = how coherent/strong the read is from available evidence (unchanged numeric value).
- evidence sufficiency = how much independent grounding supports the read (absolute, from `evidenceRichness`).
- buyer-advertised strength = derived from both; thin evidence caps the advertised band.

Failure mode addressed: the evaluator measures relative completeness inside a competition/source boundary (MLB `maxGroundedSignals == 1`, so 1-of-1 looks complete and yields a high numeric confidence), but the buyer experiences confidence across the whole artifact, where one grounded signal is thin. The gate corrects the boundary at the buyer surface without altering the internal calibration signal.

## Implementation location

`dai/apps/sports-app/src/app/analyzer/buyer-signal-summary.ts` -- the deterministic buyer projection mapper that already derives the buyer `confidenceBand` from `artifact.confidence`. This was chosen per design rule 5 (prefer the projection/UI layer where buyer-advertised strength actually lives). It is the narrowest layer that controls what buyer-facing strength means. No .NET or schema change was needed: `evidenceRichness` and `confidence` are already on the artifact DTO; the gate is a pure read-only projection.

## Evidence sufficiency derivation

`evidenceSufficiency(evidenceRichness)` returns a tier from the absolute grounded-signal count (the .NET composer sets `EvidenceRichness = GroundedSignals.Length`):
- thin: `evidenceRichness <= 1` (includes priors-only `0`)
- moderate: `evidenceRichness == 2`
- rich: `evidenceRichness >= 3`
- unknown: `evidenceRichness` absent (legacy/older artifacts)

Thresholds are the doctrine's provisional, not-outcome-calibrated values, kept in one place and easy to change.

## Buyer-advertised strength gating behavior

`gateAdvertisedStrength(rawBand, evidenceRichness)`:
- if `rawBand === 'High'` and tier is `thin` -> advertised band is capped to `Medium`, humility reason recorded.
- otherwise the band is unchanged (Medium/Low never raised or lowered; High preserved for moderate/rich).
- fails open: when sufficiency is `unknown` (no `evidenceRichness`), the band is unchanged. This avoids silently over-capping legacy artifacts; real v3 artifacts always carry `evidenceRichness`, so the gate always applies to fresh artifacts.

The gate never raises a band, never touches the lean, and never changes the raw numeric confidence (which remains on the artifact for future outcome calibration).

## Humility reason behavior

When the cap fires, `BuyerSignalSummary.advertisedStrengthReason` is set to the constant `advertised_strength_limited_by_evidence` (exported as `ADVERTISED_STRENGTH_LIMITED_BY_EVIDENCE`); it is `null` otherwise. This is factory-internal metadata available to the component, tests, and any future dev/telemetry surface. It is NOT rendered to the buyer (no template change). The buyer sees only the lower band -- plain language, no internal jargon (design rule 6).

## Tests / checks run

- TDD: added a `buildBuyerSignalSummary - evidence sufficiency band gate` describe block to `buyer-signal-summary.spec.ts` first, watched it fail to compile (`advertisedStrengthReason` not on the type -- feature missing), then implemented to green.
- 8 new tests: thin caps High->Medium (evidenceRichness 1); priors-only (0) caps; raw confidence preserved (not mutated); moderate (2) not capped; rich (3) not capped; only High is capped (Medium/Low unchanged under thin evidence); fail-open when `evidenceRichness` absent; null band with no reason when confidence absent. The existing `no mutation` test continues to guarantee confidence/posture/lean are untouched.
- Vitest suite: 38 tests pass (3 files), no regressions.
- `ng build`: bundle generation complete, exit 0.
- repo verification: `git diff --check`, added-line exact-path scan, vault non-ASCII added-line scan.

## Smoke result

Services: Docker/`devcore-sql` up, FastAPI ping 200, .NET health 200.

Full chain `POST /api/agent-runs` (MLB Orioles vs Mariners, 2026-06-09, single-signal slate): HTTP 200, `status completed`. Artifact: `artifactVersion sports_decision_artifact_v3`, `cognitiveProtocol` present, `evidenceRichness 1`, `groundedSignals ['starting_pitching']`, raw `confidence 0.75` (unchanged), `lean "Slight lean toward Orioles based on home field advantage and lineup depth."` (preserved). Applying the gate to those real inputs: rawBand `High`, evidenceTier `thin`, advertised band `Medium`, capped = true -- the buyer would now see Medium instead of High, with the lean intact. A second priors-only run (evidenceRichness 0, confidence 0.375 -> Low) was correctly left uncapped (the gate only caps High).

## Cost telemetry preservation

Cost telemetry still emits unchanged (Artifact Cost Guardrails v1 untouched): the smoke run logged `model gpt-4o-mini, totalTokens 3193, estimatedTotalCost 0.00072285, finishReason stop, status ok`. This slice added zero new model-call sites; it is a frontend projection change and makes no model calls. Two model calls were used during smoke.

## What was not changed

- no FastAPI prompt, parser, model call, or cost guardrail
- no .NET SportsEvaluator, composer, evaluator constants, or numeric confidence
- no raw confidence value, lean direction, posture, or decision logic
- no artifact schema / wire contract (purely additive frontend type field)
- no source retrieval, probe-refresh, pricing, Stripe, auth, tenant, dashboard, deployment
- no buyer copy beyond the band value itself; no buyer-facing jargon; no unsafe/tout language
- no Jera change

## Remaining gaps

- the gate is presentation-layer (Angular) only. The artifact contract and any non-Angular consumer still see only raw numeric confidence and would re-derive a band without the humility cap. Server-authoritative advertised strength (or persisting the humility reason like cost telemetry) is deferred -- recorded as ledger entry 23.
- the evidence-sufficiency thresholds (thin/moderate/rich) are provisional and not outcome-calibrated; numeric confidence recalibration still waits on Outcome Reconciliation (ledger entry 12).
- the humility reason is computed per render and not persisted/telemetered.

## Recommended next slice

- Outcome Reconciliation v1 -- the missing feedback loop that would let the numeric confidence (still preserved here) be recalibrated against real results, and would let the provisional evidence-sufficiency thresholds be validated.
- Alternatively, Advertised-Strength Server Authority v1 -- lift the gate (or the humility reason) into the artifact/contract so non-Angular consumers inherit the humility constraint, if a second consumer appears.

## Assumptions made (within scope)

- "buyer-advertised strength" maps to the existing buyer `confidenceBand`; no new buyer concept was invented.
- the gate keys on `evidenceRichness` (the explicit absolute richness field) rather than `groundedSignals.length`; for real v3 artifacts they are equal (composer sets richness = grounded count), and keying on the explicit field keeps existing band tests (which omit it) unaffected.
- fail-open on missing `evidenceRichness` (rather than capping unknown evidence) avoids regressing legacy artifacts and stays reversible.
These are local, reversible projection-layer choices that do not change product semantics beyond the approved buyer-advertised-strength gate.
