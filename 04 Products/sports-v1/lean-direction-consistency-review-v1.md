# Lean Direction Consistency Review v1

**date:** 2026-06-23
**status:** IMPLEMENTED (TDD) -- read-only containment; buyer-visible band/copy unchanged. Follows the issue surfaced by Reconciliation and Fresh Batch Validation v1 (run 722d433e).
**classification:** quality-guard containment over an existing observed-only guard. No reconciliation/confidence/advertised-strength/buyer-copy change.

## the issue

Fresh run 722d433e (MLB, Baltimore Orioles @ Los Angeles Angels, statsapi gameId 824017): structured `leanSide = home` (Angels) while the prose `lean`, summary, and counter-case all back the Orioles (away) -- "the overall backing for the Orioles is stronger given the pitching matchup." The `ArtifactDirectionConsistency` guard correctly flagged `PotentialMismatch`. This is a buyer-trust correctness issue: the artifact presented a confident structured side that its own narrative contradicts.

## root cause (verified in source + tier-1 artifact data)

Model self-inconsistency. The FastAPI analyzer (`services/agent-service/app/services/sports_analyzer.py`) has the model emit BOTH `lean` (prose) and `lean_side` (structured home/away/null); the prompt instructs them to agree (line ~128) but does not enforce it. `_parse_response` takes `lean_side` verbatim (it only has `competition`, not team names, so it cannot independently map the prose). `.NET SportsComposer` passes it through unchanged (`LeanSide: analyzerOutput.LeanSide`). For 722d433e the model emitted prose toward the Orioles but `lean_side = "home"`. So:

- NOT a structured-derivation bug, home/away mapping bug, parsing bug, or guard false positive.
- NOT a stale/local-data issue (identity and refs are correct).
- It IS category 1: model/prose contradiction, faithfully passed through by deterministic code.

## fix (containment, deterministic, smallest coherent)

New `ArtifactDirectionContainment.PresentedLeanSide(structuredLeanSide, consistency)` next to the existing guard: when the guard reports `PotentialMismatch`, the artifact read projection suppresses the structured `leanSide` to `null`; otherwise the side is preserved (`Consistent`/`Ambiguous`/`NotEvaluable`). Wired into `AgentRunsController.GetArtifact`, reusing the already-computed `ArtifactDirectionConsistency`.

- read-only: the persisted artifact, the denormalized `AgentRun.LeanSide` (which drives the reconciliation matcher), confidence, advertised strength, posture, and buyer copy are all unchanged. The mismatch stays visible via `ArtifactDirectionConsistency.Status`.
- contains BOTH the existing run 722d433e and any future run, at the contract surface, with no persisted-data change.
- no new direction system; it is one pure conditional over the proven guard.

## tests run

- TDD red -> green. New `ArtifactDirectionContainmentTests` (8): PotentialMismatch -> null; Consistent/Ambiguous/NotEvaluable preserve the side; null stays null; the **Orioles @ Angels regression** (full evaluator on the 722d433e fixture -> PotentialMismatch -> contained null); a consistent-home control keeps its side.
- Adjacent: ArtifactDirectionConsistency + AgentRunsController suites green (71).
- Declared final verification: full `DevCore.Api.Tests` 891/891.
- Live re-verify (tier-1): 722d433e now projects `leanSide = null` with `dirStatus = PotentialMismatch`, confidence 0.7 and advertisedStrength High unchanged; control 6a2d433e (Mariners, lean away) keeps `leanSide = away`, `dirStatus = Consistent`.

## is run 722d433e still unsafe/stale?

Contained at the artifact contract: it no longer presents a confident contradictory side, and the mismatch is visible. The PERSISTED `AgentRun.LeanSide` (= home) is unchanged, so if that run were reconciled, the matcher would still evaluate it as a home lean -- intentionally out of scope (rule 12: no reconciliation change). Recommendation: exclude 722d433e from calibration (it is a dev/test run with a known model-contradiction) rather than reconcile it.

## can Advertised-Strength Frontend Adoption resume?

Yes. Advertised strength is confidence/evidence-derived and was never affected. The structured `leanSide` on the contract is now contained on mismatch, so a frontend rendering server authority will not inherit a confident contradictory side. No blocker remains.

## recommended follow-ups (deferred)

- Compose-time / persist containment so the denormalized `AgentRun.LeanSide` the matcher reads is also contained on mismatch (touches the persist path; deferred to keep this slice read-only and honor rule 12).
- FastAPI prompt/parser hardening to reduce model lean-vs-lean_side contradictions at the source (the model still can emit them).
- Exclude run 722d433e from calibration cohorts.

## what was not changed

no FastAPI prompt/parser, no model call, no SportsComposer persist logic, no confidence/advertised-strength/posture, no buyer copy, no Tool Gateway/source collection/probe refresh/reconciliation/billing/auth/tenant. Existing correct artifacts preserved.
