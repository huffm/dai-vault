# Decision-Encoding Integrity -- 077A Root Cause + Settlement Guard v1

**date:** 2026-06-25
**status:** ROOT-CAUSED + GUARDED + RESOLVED. 077A excluded as ungradable (never graded). New deterministic
settlement-time integrity guard added (TDD) and verified live. Blast-radius scan run across all 184 sports runs.
No tuning: no prompt/model/confidence/posture/lean/buyer/frontend change. Corpus unchanged 61/61.
**type:** root-cause + narrow platform guard + tests in `dai`; this doc + handoff in `dai-vault`.
**skills:** dai-slice-runner + dai-skill-router gate + dai-grill-with-vault + superpowers:systematic-debugging +
superpowers:test-driven-development + superpowers:verification-before-completion + dai-test-discipline +
dai-code-reviewer (medium) + dai-agent-handoff.

**Anchor:** A run whose persisted structured lean contradicts its own artifact prose must never be graded -- a
contradictory decision has no trustworthy direction, so any verdict it receives is noise injected into calibration.
The fix is a deterministic refusal at the settlement boundary, not a model change.

## 1. Root cause (proved, classified)

Run `077a433e` (AgentRunKey 220026), Atlanta Braves @ San Diego Padres, gamePk `823284`.

Evidence chain, source-of-truth first:

1. **InputJson is correct.** `{"Competition":"mlb","HomeTeam":"San Diego Padres","AwayTeam":"Atlanta Braves"}` --
   matches MLB StatsAPI (Braves away, Padres home). **Not** a home/away identity flip.
2. **Artifact is internally contradictory.** Persisted `OutputJson`: `LeanSide="home"` (Padres) while
   `Lean="Slight lean toward Atlanta Braves based on starting pitching advantage and market consensus."`, the
   factors cite Martin Perez (the Braves' starter), and the counter-case is "the Padres could outperform" (the home
   side -> the lean is the away side). Every analytical element points to the away team; the structured field says
   home.
3. **C# is a faithful pass-through.** `SportsComposer.Compose` sets `LeanSide: analyzerOutput.LeanSide` and
   `Lean: analyzerOutput.Lean` directly -- no derivation, no team-name mapping. The denormalized
   `AgentRun.LeanSide` equals the artifact `LeanSide` (`home`). No projection or denormalization bug.
4. **Origin is the analyzer output.** `DevCore.AiClient/SportsAnalysisWire.cs` maps `lean_side` straight from the
   Python JSON. In `services/agent-service/app/services/sports_analyzer.py`, `lean_side` is **model-emitted**; the
   prompt explicitly requires it to agree with the `lean` prose ("if lean says 'Slight lean toward [home team
   name]', lean_side must be 'home'"), but `_parse_response` only clamps `lean_side` to `home/away/null` -- it does
   **not** cross-check the prose. So a model that emits prose for one side and `lean_side` for the other passes
   through unaltered.

**Classification:** **model emitted a contradictory decision.** This is NOT a serialization/projection/mapping bug
(those were each checked and are correct). Therefore, per the slice's hard boundary, **no model prompt / confidence
/ posture change was made.** The fix is a deterministic guard at the settlement boundary plus an explicit
resolution of the affected run.

## 2. Blast radius (077A is NOT isolated)

A faithful replication of `ArtifactDirectionConsistencyEvaluator` (token rules: lean > summary > factors;
distinctive tokens >= 3 chars; counter-case excluded) was run over **all 184 `sports.matchup.analysis` runs**,
cross-checked against the C# evaluator's known verdicts for 077A and 722D.

| status | count |
|---|---|
| Consistent | 56 |
| NotEvaluable | 124 (null lean / missing team refs / no team named in prose) |
| Ambiguous | 0 |
| **PotentialMismatch** | **4** |

The 4 PotentialMismatch runs (all share the same shape: `lean_side=home`, prose names the **away** team):

| run | gamePk | home / away | prose leans | settled? | recorded verdict | true (prose) read |
|---|---|---|---|---|---|---|
| 077A433E | 823284 | padres / braves | Braves (away) | **no** | -- | n/a (excluded) |
| 722D433E | (Angels game) | los-angeles-angels / baltimore-orioles | Orioles (away) | no | -- | -- |
| **BDDE423E** | 822889 | texas-rangers / minnesota-twins | Twins (away) | **yes** | incorrect (away_win, lean home) | should be **correct** (prose=away, away won) -> false-negative grade |
| **01AA433E** | 824910 | atlanta-braves / milwaukee-brewers | Brewers (away) | **yes** | **correct** (home_win, lean home) | should be **incorrect** (prose=away, home won) -> **false-positive in corpus** |

**Two already-settled runs (BDDE423E, 01AA433E) were graded on a contradictory lean** -- both verdicts are wrong
relative to the artifact's actual prose read (one false-correct, one false-incorrect). They contaminate the
calibration corpus in both directions. They are flagged here for a remediation decision; **they were not mutated in
this slice** (re-grading/excluding settled corpus rows is a deliberate, separately-scoped data action, and the
boundary forbids silent overwrites).

## 3. Guard added (deterministic, TDD, reuses existing evaluator)

The codebase already had `ArtifactDirectionConsistencyEvaluator` (flags `PotentialMismatch`) and
`ArtifactDirectionContainment` (nulls the side on the **read** surface). The gap: the **settlement write path**
graded the denormalized `LeanSide` without consulting either. This slice closes that gap.

- **`DevCore.Api/AgentRuns/SettlementDirectionIntegrity.cs`** (new, pure): wraps the evaluator and blocks settlement
  iff `Status == PotentialMismatch`. Fail-soft: Consistent / Ambiguous / NotEvaluable settle normally (a null lean
  is NotEvaluable -> still graded inconclusive by `RunEvaluator`, never a side).
- **`AgentRunsController`**: both write paths (`POST /{id}/outcome` and `POST /reconcile` SingleMatch) now call one
  shared guard `DirectionIntegrityRefusalOrNull(...)` after the idempotency check and before
  `AddOutcomeAndEvaluation`. On a clear contradiction they return **422** with the integrity diagnostic
  (`status`, `reason`, `structuredLeanSide`, `detectedProseSides`, `warnings`) and write nothing. The guard keys on
  the denormalized `run.LeanSide` -- the exact value `RunEvaluator` grades -- so the refusal is aligned with what
  would have been written.

The guard does NOT mutate any artifact, `LeanSide`, confidence, posture, advertised strength, or buyer copy.

### Tests (red observed before green)

- `SettlementDirectionIntegrityTests` (5, pure): contradiction blocks; consistent allows; null lean allows
  (NotEvaluable); missing team refs allows; ambiguous (both teams named) allows.
- `AgentRunsControllerTests` (4, integration): contradictory run -> 422 and writes nothing; consistent run -> 201;
  null-lean run -> 201 + inconclusive (never a side); two runs sharing a gamePk are guarded independently by
  agentRunId (consistent one settles, contradictory one refused) -- proves the guard is per-run.
- Verification: targeted ladder green; **final verification full DevCore.Api.Tests 903/0**; `git diff --check`
  clean; added comments lowercase ascii. Code review (medium) run; findings triaged (sec 5).

## 4. 077A resolution (ungradable -> excluded; never graded)

Because the true encoded direction cannot be proved (the model contradicted itself), 077A is **not settleable**. It
was **explicitly excluded** as an integrity failure via `POST /api/agent-runs/{id}/exclude` with reason `invalid`
("malformed/erroneous run that must not be evaluated"). Live evidence:

- `POST /{id}/outcome` on 077A -> **422** `prose_points_to_opposite_side_in_lean` (the deployed guard refuses it).
- `POST /{id}/exclude {reason:"invalid"}` -> **200**.
- DB after: `LeanSide=home` (UNCHANGED -- the contradiction is preserved, not hidden), `ExclusionReason=invalid`,
  outcome=0 / eval=0 (never graded). Corpus unchanged **61/61**.

077A is out of matcher/calibration selection and can never be graded while the contradiction stands.

## 5. Known limitations (documented, not silently accepted)

- **False-positive (conservative):** the evaluator decides on the first prose section that names any team; a run
  that leans the home side in words ("back the home side") while only naming the away team in that sentence would be
  flagged PotentialMismatch and refused. The failure mode is safe -- a missed grade plus a 422 diagnostic, never a
  wrong grade -- and the empirical corpus shows zero such cases (all 56 Consistent runs name their leaned team).
- **False-negative on partial artifacts:** the controller builds the guard input via `TryDeserializeArtifact`,
  which returns false when `Summary` or `Factors` is null; such a run fail-softs to "allowed." All in-scope v3
  artifacts populate both fields, and the blast-radius scan (which parses fields directly, without that gate) is the
  authoritative full-corpus check -- it found only the 4 mismatches above, all fully-formed. The periodic
  blast-radius scan is the backstop for this gap.

## 6. Effect on cohort v3 sequencing

- **Cohort v3 capture is SAFE to proceed now:** the settlement guard prevents any future contradictory run from
  being graded, so new slates cannot be contaminated the way BDDE/01AA were.
- **Before the next Calibration Read (v3):** remediate the two already-settled mismatches (BDDE423E, 01AA433E) --
  re-evaluate them against their true prose read or exclude them -- so the pooled corpus is clean. The read should
  not pool a known false-correct (01AA) with valid samples.

## 7. Recommended next slices

1. **Mismatch-Remediation v1:** decide and apply the fix for BDDE423E + 01AA433E (re-grade vs exclude), documented;
   then the corpus is clean for Read v3.
2. **Analyzer Lean-Agreement Hardening v1 (generation side, separate lane):** make `_parse_response` (or a
   post-analyze check) detect `lean_side` vs prose disagreement at generation time and degrade to null rather than
   emit a contradiction. This is the upstream fix; it is a generation-behavior change and must be scoped on its own
   (not "tuning" of confidence/lean, but a correctness guard).
3. **Directional-Contrast Cohort Capture v3 + Market Baseline v2:** proceed; the slate need is unchanged.

## 8. What did not change

No model/prompt, confidence, posture, lean, advertised strength, buyer copy, frontend, or reconciliation-evaluation
math change. No artifact or `LeanSide` mutation. No settled outcome/evaluation row was altered or deleted. Corpus
61/61. The only code added is the deterministic guard + its tests; the only data change is 077A's `ExclusionReason`.
