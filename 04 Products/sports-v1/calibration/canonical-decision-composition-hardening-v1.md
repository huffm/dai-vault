# Canonical Decision Composition Hardening v1

**date:** 2026-06-28
**status:** IMPLEMENTED + VERIFIED. The analyzer composition boundary now validates that the structured lean (the
canonical decision) and the narrative prose agree, persists that verdict on the artifact, and fails closed: a clear
structured-vs-prose contradiction is born settlement-ineligible (ExclusionReason=invalid). The settlement-time 422
guard is preserved as the independent safety net. No prompt/model/confidence/posture/buyer/advertised-strength
change. No Cohort v4 settlement; 0 outcomes written.
**type:** composition-boundary integrity slice. dai-slice-runner + dai-skill-router gate + systematic-debugging +
test-driven-development + dai-test-discipline + verification-before-completion + dai-grill-with-vault.
**anchor:** A run whose persisted lean contradicts its own artifact prose has no trustworthy direction (the 077A
precedent). Until now that was caught only at settlement (422) -- after the contradictory artifact was already
persisted with a confident structured side. This slice moves enforcement to the composition boundary so the class of
artifact cannot be persisted as settlement-eligible in the first place.

See also: [[mismatch-remediation-v1]] (denominator + historical-mismatch doctrine), [[directional-contrast-cohort-v4-market-baseline-v3]]
(the slate that exposed run 4f37433e), [[directional-contrast-cohort-v3-settlement-v1]], [[calibration-assessment-v3]].

## 1. Problem observed

The structured decision (`AgentRun.LeanSide` / analyzer `lean_side`) and the buyer-visible narrative prose could
diverge. The analyzer prompt already instructs "lean_side must agree with lean" (`sports_analyzer.py:128`), but that
is an instruction to a model, not an enforced invariant -- the model still produced a contradiction. The only
protection was the settlement-time `SettlementDirectionIntegrity` guard (422), which fires *after* a contradictory
artifact has already been composed, persisted, and denormalized with a confident structured side.

## 2. Example mismatch from Cohort v4

Run **4f37433e** -- Philadelphia Phillies @ New York Mets (gamePk 823608):

- structured `LeanSide = home` (Mets); prose `lean` = *"Slight lean toward Phillies based on starting pitching
  advantage and market support"* (Phillies = away); `summary` supports the Phillies.
- guard verdict: `status=PotentialMismatch`, `reason=prose_points_to_opposite_side_in_lean`,
  `structuredLeanSide=home`, `detectedProseSides=[away]`.
- Same class as the historically-remediated 077A / BDDE423E / 01AA433E and the unsettled 722D433E.

## 3. Root-cause analysis

The pipeline had no single point where the canonical decision and its prose were checked for agreement before
persistence. Detection logic existed (`ArtifactDirectionConsistencyEvaluator`) but ran only read-only (artifact read,
settlement guard). Composition (`SportsComposer`) assembled the artifact from the analyzer output and persisted it
verbatim -- a divergent `lean_side` vs prose was persisted silently and only stopped later at settlement. The bug is
therefore a composition-boundary bug, not merely a settlement-validation gap.

## 4. Canonical decision composition rule

The analyzer call is the single source object: it produces both the structured `lean_side` and the prose in one
response. The composer consumes that one object. The invariant is now enforced at composition:

- the structured lean is the canonical decision;
- the prose (lean sentence > summary > factors) must support that same side;
- if the prose points only to the opposite side -> `PotentialMismatch` -> fail closed.

The prose is treated as a projection/explanation of the canonical decision, never a second independent decision.
The fix is composition-level enforcement, not new NLP -- it reuses the existing deterministic, fail-soft
`ArtifactDirectionConsistencyEvaluator` (lean/summary/factors weighted; counter-case excluded as a direction source).

## 5. Implementation point

All in `dai` (.NET), at the composition + persistence boundary; no Python change (the detector already exists in
.NET; duplicating prose parsing in Python was explicitly avoided):

- `AgentRuns/SportsComposer.cs` -- `Compose` computes the lean-agreement verdict once from the analyzer output
  (`LeanSide`, `Lean`, `Summary`, `Factors`) + canonical matchup identity (`artifact.Input.HomeTeam/AwayTeam`) and
  passes it into the result.
- `AgentRuns/AgentRunContracts.cs` -- `AgentRunExecutionResult` gains a persisted, nullable `LeanAgreement`
  (`ArtifactDirectionConsistency`) field. Back-compat: older records / analyze-failure runs deserialize to null.
- `AgentRuns/CompositionEligibility.cs` (new) -- pure helper mapping a `PotentialMismatch` verdict to
  `RunExclusionReasons.Invalid` (the same audited eligibility reason used for historical contradictions).
- `Controllers/AgentRunsController.cs` -- on the create/persist path, after identity capture, a composition mismatch
  sets `run.ExclusionReason = invalid` (born settlement-ineligible) and logs a warning. The artifact's lean, prose,
  and confidence are persisted unchanged.

## 6. LeanAgreement / integrity status semantics

`LeanAgreement.Status` (from the existing evaluator):

- `Consistent` -- prose supports the structured side. Eligible.
- `PotentialMismatch` -- prose points to the opposite side only. **Settlement-ineligible (fail closed).**
- `Ambiguous` -- prose names both sides. Eligible (structured side stays authoritative).
- `NotEvaluable` -- null lean, or insufficient team/prose text. Eligible (a null lean grades inconclusive, never a
  side).

The verdict is persisted on the artifact, so the structured/prose agreement is recorded at the boundary (not silent)
and is queryable for the denominator.

## 7. Settlement behavior

Two layers, fail-closed:

1. **Composition (new):** `PotentialMismatch` -> `ExclusionReason=invalid` at birth -> the reconciliation matcher
   (`OutcomeReconciliationService.MatchAsync`, selects `ExclusionReason IS NULL`) never selects it; no outcome is
   ever written via the reconcile path.
2. **Settlement guard (preserved):** `SettlementDirectionIntegrity` still returns 422 on any `PotentialMismatch`
   reaching `POST /outcome` or `/reconcile`, independent of `ExclusionReason`. Verified live: a `POST /outcome` to
   4f37433e returns 422 and writes nothing.

## 8. Denominator behavior

The valid-calibration denominator is unchanged in definition: **settled AND `ExclusionReason IS NULL`**. A
composition mismatch is born `ExclusionReason=invalid`, so it is excluded by the existing filter -- the matcher and
every conformant read already honor it ([[mismatch-remediation-v1]] sec 5). Current valid denominator = **74**
(unaffected by this slice; 4f37433e was unsettled and never counted).

## 9. Tests and verification

- New `DevCore.Api.Tests/AgentRuns/CanonicalDecisionCompositionTests.cs`: canonical home->consistent, canonical
  away->consistent, null->no invented side (NotEvaluable), structured-home/prose-away->PotentialMismatch,
  structured-away/prose-home->PotentialMismatch, the modeled 4f37433e shape->mismatch + born `invalid`, eligibility
  mapping for every status (+null), and settlement-guard parity on the same input.
- **Targeted suite: 72/72 passed** (composition + composer + settlement guard + consistency evaluator + matcher).
- **Full `DevCore.Api.Tests`: 940/940 passed, 0 failed** -- no regression from the new field / composer change.
- Commands: `dotnet test .\DevCore.Api.Tests\DevCore.Api.Tests.csproj --filter "..."` then unfiltered.
- Live DB/API verification (devcore-sql + DevCore.Api from the hardened build): 4f37433e
  `ExclusionReason=invalid`; settlement attempt -> 422 (nothing written); v4 = 14 clean (ExclusionReason NULL) + 1
  excluded; v4 outcomes/evaluations written = 0; corpus outcomes/evals = 76 (unchanged); valid denominator = 74;
  duplicate outcomes = 0.

## 10. Historical artifact policy

Run **4f37433e** (already persisted before this fix) was handled per the historical-mismatch policy without editing
the artifact: the original lean, prose, and confidence are preserved; it is classified `PotentialMismatch`,
settlement is blocked (422), and it was excluded from the valid denominator via the audited
`POST /api/agent-runs/{id}/exclude {"reason":"invalid"}` mechanism (the same path used for BDDE423E/01AA433E in
[[mismatch-remediation-v1]]). No prose or structured lean was rewritten; no verdict was created or overwritten. This
supersedes the interim "left as-is, flagged" disposition recorded in
[[directional-contrast-cohort-v4-market-baseline-v3]] sec 5 -- with the composition hardening in place, the historical
run is now aligned to how new mismatches are handled (born excluded).

## 11. Remaining risks

- Enforcement is in .NET composition; the Python analyzer can still *emit* a divergent `lean_side`/prose pair -- it
  is now caught and quarantined deterministically, but not prevented at generation. A future generation-side guard
  (degrade `lean_side` to null when it disagrees with the prose) would stop production at the source
  ([[mismatch-remediation-v1]] sec 9 rec 1).
- The detector is token-based; very short or unusual team strings can yield `NotEvaluable` (fail-soft -> eligible).
  This is intentional (never block on insufficient evidence) but means detection is best-effort, not exhaustive.
- 722D433E (a separate unsettled mismatch) was not touched by this slice; it remains settlement-blocked by the guard
  and is a candidate for the same audited exclusion.

## 12. Recommended next slice

**Analyzer Lean-Agreement Hardening v1 (generation side):** in `sports_analyzer.py` `_parse_response`, degrade
`lean_side` to null when a deterministic check finds it disagrees with the prose, so contradictions stop being
produced at the source -- complementing this composition-side fail-closed. Separately: settle Cohort v4 after
2026-06-29 finals (the 14 clean runs; 4f37433e is excluded and 422-blocked); continue toward >=3 settled
directional-contrast slates before Calibration Assessment v4. Do not tune.
