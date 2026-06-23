# Advertised-Strength Server Authority v1

**date:** 2026-06-23
**status:** IMPLEMENTED (TDD). server-derived read-only projection; buyer-visible band unchanged. no confidence/lean/posture mutation, no buyer copy change, no Tool Gateway/source/probe/reconciliation/billing/auth/tenant change.
**classification:** read-model contract addition. first slice run through `dai-slice-runner` + `dai-code-reviewer`.

## problem

Buyer-advertised strength (the confidence band AFTER the evidence-sufficiency humility cap) was a frontend-only concern: `gateAdvertisedStrength` in `apps/sports-app/src/app/analyzer/buyer-signal-summary.ts` capped `High + thin` to `Medium`. The artifact contract exposed raw `Confidence` and `EvidenceRichness` but no advertised strength, so any non-Angular consumer would re-derive a band without the humility cap. The Evidence-Sufficiency Band Gate v1 doc named this gap (ledger entry 23). This is the lift recommended by Source Depth and Evidence Sufficiency Doctrine v1.

## what shipped

- `DevCore.Api/AgentRuns/AdvertisedStrength.cs` (new, pure): `AdvertisedStrengthDeriver.Derive(double? confidence, int? evidenceRichness)` returns `AdvertisedStrength(Band, RawBand, Reason)`.
  - raw band thresholds reproduce the existing buyer projection exactly: `>= 0.70` High, `>= 0.45` Medium, else Low; null confidence -> null band. **the buyer-visible band is therefore unchanged by the lift.** (the evaluator's internal `ConfidenceBand` uses a separate 0.50 medium threshold and is not the buyer-advertised band.)
  - evidence-sufficiency cap: `High + evidenceRichness <= 1` (thin, includes priors-only 0) -> capped to `Medium`, `Reason = advertised_strength_limited_by_evidence` (matches the frontend constant; internal metadata, never buyer copy).
  - fails open when `evidenceRichness` is absent (legacy artifacts) -> band unchanged. never raises a band; never touches Medium/Low; never mutates confidence/lean/posture.
- `AgentRunArtifactDto` gains a trailing optional `AdvertisedStrength? AdvertisedStrength` (no positional shift at other construction sites).
- `AgentRunsController.GetArtifact` derives it read-only beside `SourceSufficiency`, from `artifact.Confidence` + `artifact.EvidenceRichness`. `Confidence` is passed through unchanged.

## two-axis fidelity

Confidence (internal coherence) stays the numeric value, unchanged and provisional. Evidence sufficiency (absolute grounding) stays `EvidenceRichness`. Buyer-advertised strength derives from both; thin evidence cannot advertise High. The server now owns this derivation; the contract is the authority a non-Angular consumer reads.

## verification

- TDD red -> green: `AdvertisedStrengthTests.cs` (13 cases) -- buyer thresholds at boundaries (0.70/0.45/0.4499), thin caps High->Medium, priors-only (0) caps, moderate(2)/rich(3) uncapped, only High capped (Medium/Low unchanged), fail-open on null richness, null confidence -> null band.
- Adjacent: `AgentRunsControllerTests` 54/54 (artifact endpoint uses the changed DTO ctor).
- Declared final verification: full `DevCore.Api.Tests` 884/884.
- `dai-code-reviewer`: approve with notes (no blocking issues).

## deferred (documented)

- **Advertised-Strength Frontend Adoption v1** (recommended next): render `artifact.advertisedStrength` and retire the duplicate frontend `gateAdvertisedStrength`/`confidenceBand` computation, plus an `/artifact` endpoint test asserting the field is surfaced and `Confidence` is unchanged. Until then the frontend computes its own identical band -- a divergence risk if one threshold changes and not the other.
- Numeric confidence recalibration stays gated on Outcome Reconciliation (ledger entry 12).
- Buyer-visible depth/strength wording unchanged.

## what was not changed

no FastAPI prompt/parser/model call, no `SportsEvaluator` constants or numeric confidence, no lean/posture, no buyer copy, no Angular, no schema/migration, no Tool Gateway/source retrieval/probe-refresh/reconciliation/pricing/auth/tenant. Frontend untouched.
