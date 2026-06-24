# Advertised-Strength Frontend Adoption v1

**date:** 2026-06-24
**status:** IMPLEMENTED (TDD). Angular frontend + tests. Buyer-visible band unchanged. No confidence/lean/posture/buyer-copy change, no backend change, no Tool Gateway/source/probe/reconciliation/billing/auth/tenant change.
**classification:** frontend projection adoption. closes the dual authority opened by Advertised-Strength Server Authority v1.

## problem

Advertised-Strength Server Authority v1 (`a499b80`) lifted buyer-advertised strength (the confidence band AFTER the evidence-sufficiency humility cap) into the artifact read-model contract, but the Angular buyer path still re-derived its own identical band in `apps/sports-app/src/app/analyzer/buyer-signal-summary.ts` (`confidenceBand` + `gateAdvertisedStrength`). That left two authorities for the same value: if one threshold changed and not the other, the buyer band would diverge. The Server Authority note named this slice as the recommended fix.

## what shipped

Three files, all under `apps/sports-app`:

- `core/models/agent-run.model.ts`
  - new optional `AdvertisedStrengthDto` (`band` / `rawBand` / `reason`, all optional strings -- camelCase wire shape of the backend `AdvertisedStrength` record).
  - `AgentRunArtifactDto` gains optional `advertisedStrength`.
- `analyzer/buyer-signal-summary.ts`
  - `serverAdvertisedStrength(artifact)` -- **PRIMARY AUTHORITY.** returns the server band + reason when the artifact carries `advertisedStrength`, narrowing the band through `asBuyerBand` so an unexpected token can never leak onto the buyer surface. returns null only when the field is entirely absent (legacy/older artifacts).
  - `fallbackAdvertisedStrength(artifact)` -- **FALLBACK ONLY.** the prior `confidenceBand` + `gateAdvertisedStrength` derivation, now clearly demoted; mirrors `AdvertisedStrengthDeriver.Derive` (same 0.70/0.45 thresholds and the same High + thin -> Medium cap). used only when the server field is missing.
  - `buildBuyerSignalSummary` selects `serverAdvertisedStrength(artifact) ?? fallbackAdvertisedStrength(artifact)`.
- `analyzer/buyer-signal-summary.spec.ts` -- +6 tests.

The duplicate frontend authority is **demoted to a compatibility fallback**, not deleted, so artifacts produced before the server lift still render the same buyer band.

## buyer-visible band unchanged

The server reproduces the exact derivation the frontend used (`AdvertisedStrength.cs` thresholds equal `buyer-signal-summary.ts` thresholds), so for equivalent inputs the rendered band is identical whether it comes from the server path or the fallback path. Confidence, lean, posture, and buyer copy are untouched; the existing no-mutation test stays green.

## verification

- TDD red -> green. Scoped `npm test -- --watch=false --include="**/buyer-signal-summary.spec.ts"` 29/29 (was 23).
- Tests prove: server field present -> frontend uses it (server `Medium` overrides a local-uncapped `High`); server field missing -> fallback preserves legacy behavior; `High` + thin does not render `High` on the fallback path; null / unexpected server band -> null (defensive).
- Declared final verification: full sports-app suite 65/65; `npx ng build` clean.
- Backend tests not run (no backend change).
- `dai-code-reviewer`: approve (no blocking issues; non-blocking: `rawBand` is typed for contract parity but not consumed by the buyer surface).

## LeanSide Direction Consistency Fix v1 -- intentionally unchanged

LeanSide Direction Consistency Fix v1 (`e4a00de`) remains untouched. Advertised strength is confidence/evidence-derived and was never affected by the structured-`leanSide` containment; confirmed in source and in `lean-direction-consistency-review-v1.md` ("Advertised strength ... was never affected ... No blocker remains"). No compile/type issue required model alignment, so rule 12 was honored without exception.

## deferred (documented)

- Optional dev-only surfacing of the `advertised_strength_limited_by_evidence` humility reason on the artifact inspection page (inspection metadata, not buyer copy).
- An `/artifact` endpoint integration test asserting the field is surfaced and `Confidence` is unchanged stays a backend concern (named in the Server Authority note).

## what was not changed

no backend advertised-strength derivation (`AdvertisedStrength.cs`, `AgentRunsController.GetArtifact`), no FastAPI/model, no `SportsEvaluator`/numeric confidence, no lean/posture, no buyer copy, no schema/migration, no Tool Gateway/source collection/probe refresh/reconciliation/billing/auth/tenant. No semantic/cloud graphify; `graphify-out/` not committed.
