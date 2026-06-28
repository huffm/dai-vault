# Analyzer Generation-Side Lean Agreement Hardening v1

**date:** 2026-06-28
**status:** IMPLEMENTED + VERIFIED. The Python analyzer now fails closed at the generation boundary: when the model's
structured `lean_side` contradicts its own prose, `lean_side` is degraded to null (NoLean) before the response leaves
`sports_analyzer.py`. Deterministic; no model/prompt/confidence change; contract-safe (no new fields). Complements the
.NET composition-boundary hardening, which remains the second safety net.
**type:** source-side containment slice. dai-slice-runner + dai-skill-router gate + systematic-debugging +
test-driven-development + dai-test-discipline + verification-before-completion + dai-grill-with-vault.
**anchor:** the .NET boundary quarantines contradictions *after* analyzer output; this slice prevents a contradictory,
valid-looking structured lean from *leaving* the analyzer in the first place.

See also: [[canonical-decision-composition-hardening-v1]] (the .NET downstream net), [[mismatch-remediation-v1]]
(denominator + mismatch doctrine), [[directional-contrast-cohort-v4-market-baseline-v3]] (run 4f37433e).

## 1. Problem observed

The known failure shape (run 4f37433e, Phillies@Mets): the model returned `lean_side="home"` while its prose
(`lean`/`summary`) leaned the away team (Phillies). The analyzer emitted that as a normal, valid `SportsAnalysisResponse`
with a confident structured side. Nothing in Python caught it; .NET persisted the contradiction (pre-hardening), and
settlement caught it later with a 422.

## 2. What was wrong (plain English)

`_parse_response` in `services/agent-service/app/services/sports_analyzer.py` reads `lean_side` and the prose from the
**same** model JSON but trusts them **independently**. The structured side is taken at line ~442
(`raw_side = parsed.get("lean_side")`; clamped only to home/away/null), and the prose `lean` is taken separately at
line ~436. There was **no check that the two agree**, and `_parse_response` did not even receive the team names needed
to tell which side the prose names. The system prompt does instruct agreement ("lean_side must agree with lean",
`sports_analyzer.py:128`), but a prompt is guidance to a model, not enforcement -- the model violated it and the parser
trusted the structured value anyway. So the root cause is **parser trust without validation**, not a duplicated
decision path and not (only) weak prompt guidance.

The prior .NET fix ([[canonical-decision-composition-hardening-v1]]) was downstream only: it computed `LeanAgreement`
at compose time and marked a mismatch `ExclusionReason=invalid` -- correct, but it acts *after* the analyzer has
already produced a confident contradictory decision object. This slice moves containment to the source: the analyzer
no longer emits a valid structured side when its own prose contradicts it. Remaining risk: detection is token-based and
fail-soft (see sec 10).

## 3. Exact root cause

`_parse_response` trusts `parsed["lean_side"]` independently of the prose; no agreement validation existed on the
generation side, and the function lacked the home/away team names required to detect the prose-supported side.

## 4. Code path affected

`services/agent-service/app/services/sports_analyzer.py`:
- `_parse_response(raw, competition)` -- still parses as before (unchanged; malformed JSON still raises `ValueError`).
- new pure helpers: `_tokenize`, `_distinctive_tokens`, `_contains_word`, `_prose_supported_side`, and
  `_contain_lean_agreement`.
- `_call_model(system, user_msg, competition, home_team, away_team)` -- gains the two team-name params and applies
  `_contain_lean_agreement` to the parsed response before returning. This is the single enforcement point for all
  three sports (the `analyze_football` / `analyze_basketball` / `analyze_mlb` call sites now pass the team names they
  already hold).

## 5. Source-side containment rule

After parsing, determine the prose-supported side deterministically (distinctive team tokens, length >= 3, minus
shared tokens; weighting `lean` > `summary` > `factors`; the highest-weight section that names a side decides; a
section naming both teams is ambiguous -> None). If `lean_side` is home/away and the prose clearly supports the
opposite side, **degrade `lean_side` to null (NoLean)** and log an integrity warning. Everything else is preserved.
The logic mirrors the .NET `ArtifactDirectionConsistencyEvaluator` so the two boundaries agree.

## 6. Behavior before

`lean_side` was emitted exactly as the model returned it (clamped only to home/away/null). A structured side that
contradicted the prose left the analyzer as a valid directional decision.

## 7. Behavior after

- structured side agrees with prose -> unchanged.
- structured side contradicts prose (prose clearly names the opposite team) -> `lean_side` degraded to null;
  `lean`/`summary`/`factors`/`confidence` preserved; a warning is logged. Downstream, a null lean grades inconclusive
  and never settles to a side; the .NET composition boundary sees a null lean (NotEvaluable), not a contradiction.
- null `lean_side` -> unchanged (no side invented).
- ambiguous prose (both teams named) or prose naming neither team -> unchanged (structured side stays authoritative;
  no false contradiction).

Confidence is intentionally NOT changed (no confidence tuning; the .NET evaluate step owns final confidence, and a
null lean is inconclusive regardless).

## 8. Test coverage

New `services/agent-service/tests/test_lean_agreement_containment.py` (11 tests, deterministic, no model calls):
structured-home+prose-home keeps home; structured-away+prose-away keeps away; structured-home+prose-away -> null;
structured-away+prose-home -> null; modeled 4f37433e shape -> null (prose + confidence preserved); null stays null;
ambiguous (both teams) no false contradiction; prose naming neither team unchanged; and three `_prose_supported_side`
unit cases.

Verification commands + results:
- `python -m pytest tests/test_lean_agreement_containment.py -q` -> **11 passed**.
- `python -m pytest tests/test_sports_analyzer.py -q` -> **109 passed** (8 `_call_model` test doubles updated to the
  new signature -- mechanical, no behavior weakened).
- `python -m pytest -q` (full agent-service suite) -> **126 passed, 0 failed**.
- `.NET` (no source change this slice; safety net confirmed): `dotnet test --filter
  "FullyQualifiedName~CanonicalDecisionComposition|FullyQualifiedName~SettlementDirectionIntegrity"` -> **17 passed**.

## 9. Relationship to .NET composition-boundary hardening

Two independent, deterministic boundaries sharing the same detection semantics:
- **Generation side (this slice):** the analyzer degrades a contradictory `lean_side` to null before output -> no
  confident contradictory decision is produced.
- **Composition side ([[canonical-decision-composition-hardening-v1]]):** if a contradiction somehow survives Python,
  `SportsComposer` persists `LeanAgreement=PotentialMismatch` and the run is born `ExclusionReason=invalid`; the 422
  `SettlementDirectionIntegrity` guard blocks settlement.
Defense in depth: the source net prevents, the composition net quarantines. Confirmed intact (17 .NET tests green).

## 10. Remaining risks

- Detection is token-based and fail-soft: very short or unusual team strings, or prose that refers to a side without
  naming the team ("the home side"), yield NotEvaluable and the structured side is left as-is. Intentional (never
  block on insufficient evidence), but means containment is best-effort, not exhaustive -- which is exactly why the
  .NET net is retained.
- The model can still *generate* a contradictory pair; this slice contains it, it does not stop the model from
  reasoning inconsistently. A future prompt/schema iteration could reduce the generation rate (out of scope: no
  prompt tuning this slice).
- Degrading to null converts a contradictory run into a legitimate abstention (null lean). This is the safe outcome
  but means the run contributes no directional signal -- acceptable and consistent with doctrine.

## 11. Recommended next slice

Optional: a small observability counter/log aggregation for how often generation-side containment fires (a calibration
signal on model self-consistency), still read-only and non-tuning. Otherwise resume the paused lanes: settle Cohort v4
after 2026-06-29 finals (14 clean runs; 4f37433e excluded + 422-blocked); pool >=3 settled directional-contrast slates
before Calibration Assessment v4. Do not tune.
