---
title: "HANDOFF: Market Attribution Fidelity Guard v1 -- SHIPPED (2026-07-07)"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- guard live on /rows; 1069 tests green; 6/6 known accidentals classified; 4 true deliberate divergences discovered in legacy rows"
project: "DAI"
slice: "Market Attribution Fidelity Guard v1"
related:
  - "06 Execution/patterns/market-attribution-fidelity-guard-v1.md"
  - "06 Execution/reports/market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md"
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
---

# HANDOFF: market attribution fidelity guard v1 (2026-07-07)

## 1. objective

implement the deterministic prose-vs-staged-market fidelity guard (debug report section
10 spec): PASS/FAIL/UNCLEAR + divergence interpretation (aligned/deliberate/accidental/
unclear), surfaced additively on /rows, never blocking settlement, never touching gate-4
counts, TDD.

## 2. outcome

SHIPPED. `DevCore.Api/AgentRuns/MarketAttributionFidelity.cs` (pure evaluator + the
CountsAsCandidateEdge ledger rule) + shared TeamProseTokens extraction from the direction
guard + 3 additive /rows fields (attributionFidelityStatus, attributionFidelityReason,
divergenceInterpretation). **17 new tests, full suite 1069/1069.** live verification on
285 rows: **all 6 taxonomy accidental divergences classify AccidentalDivergence; both
agree-row inversions (824012 + newly found 824011) FAIL; and the guard discovered 4 TRUE
deliberate divergences in legacy rows outside the audit corpus** (823284, 823608 -- both
excluded `invalid`; 823281 x2 -- active but unsettled AND a new duplicate-active hazard),
each manually verified against team refs (prose named the away team the staged consensus
favored while leaning home). the settled VALID divergence ledger remains 100% accidental
-- the taxonomy conclusion stands. baseline for the future prompt-hardening slice:
Pass 72 / FAIL 10 / Unclear 203 (unclear mass = legacy rows without market snapshots).

## 3. repo state before / after

- dai before: main @ `98b3306`, 0/0, phantom csproj. after: guard commit pushed (sha in
  closeout), 0/0, phantom untouched.
- dai-vault before: `4053e5c`, 0/0, known untracked noise. after: pattern doc + this
  handoff committed and pushed.

## 4. services used

devcore-sql (up). DevCore.Api :5007 stopped for the build (it locked the DLLs), then
restarted from the new build and used for live /rows + /artifact verification.
agent-service NEVER started; zero model calls; zero paid calls; statsapi untouched.

## 5. work performed

- skills gate (TDD + test-discipline + code-reviewer selected) -> baselines clean.
- read the analogous guard (ArtifactDirectionConsistencyEvaluator -- reused its
  distinctive-token side identification via a TeamProseTokens extraction), the /rows
  exporter (prose-free row doctrine test found BEFORE wiring -- shaped the design), the
  artifact DTO (all prose fields already deserialized).
- TDD: 15 evaluator tests written first (real 823036/824820 prose fixtures, deliberate
  synthetic, 824012-style agree inversion, 824175-style both-directions, missing-input,
  shared-nickname red-sox/white-sox, determinism, ledger rule) -> red -> implement ->
  two real defects caught and fixed by the cycle: (1) sentence-split on "St." (st. louis)
  produced a fragment that defeated the self-referential-clause exclusion -> lookbehind
  fix; (2) self-referential clauses ("this read leans X ... the market signal")
  registered as market claims -> exclusion rule. -> 15/15.
- exporter wiring: 3 additive trailing fields computed in Shape from persisted fields +
  already-deserialized prose; raw evidence clause deliberately NOT projected (prose-free
  row doctrine); 2 exporter tests (823036-shaped end-to-end incl.
  marketAgreement-untouched assertion; no-snapshot row -> Unclear).
- full suite 1069/1069 -> api restarted -> live verification incl. manual team-ref
  confirmation of all 4 DeliberateDivergence rows -> dai-code-reviewer review (approve)
  -> pattern doc -> commits + pushes.

## 6. files changed

dai (one commit):
- `platform/dotnet/DevCore.Api/AgentRuns/MarketAttributionFidelity.cs` (new)
- `platform/dotnet/DevCore.Api/AgentRuns/ArtifactDirectionConsistency.cs` (tokenization
  extracted to internal TeamProseTokens; behavior identical, existing tests green)
- `platform/dotnet/DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` (3 additive
  trailing row fields + evaluation in Shape)
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/MarketAttributionFidelityTests.cs` (new, 15)
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/PromptRouteCalibrationExporterTests.cs` (+2)
dai-vault (one commit):
- `06 Execution/patterns/market-attribution-fidelity-guard-v1.md` (new)
- this handoff.

## 7. db writes / side effects

0 db writes (guard is derive-on-read; in-memory EF in tests only). side effects:
DevCore.Api restarted on the new build (left running); the 2026-07-07 cohort untouched
(still 6/6 unreconciled).

## 8. paid calls / cost

0 paid model calls, $0.00.

## 9. validation proof

- tests: 1069/1069 full suite (17 new: 15 evaluator + 2 exporter); existing
  direction-integrity, calibration metrics, and prose-free row tests all green.
- live /rows (285): 6/6 known accidentals -> AccidentalDivergence with reason
  `prose_claims_home_but_staged_consensus_is_away`; 824012 + 824011 FAIL on agree rows;
  4 deliberate classifications manually verified against awayTeamRef (braves/phillies/
  dodgers all away = staged side named in prose).
- invariants: marketAgreement values unchanged (asserted in test + live spot check);
  gate-4 criterion file untouched; settlement path never reads the new fields;
  /metrics calculator untouched (values consistent with the +6 settlement this morning);
  no model call anywhere in the guard (pure function; test-asserted no service
  dependency); registry flags untouched; no prompts/templates changed.

## 10. what did not change

prompts, templates (the overlay hazard remains until Prompt Market Context Hardening
v1), confidence, models, gate-4 criterion/thresholds/counts, persisted marketAgreement,
settlement eligibility and /reconcile, buyer copy, registry default-off, the unsettled
2026-07-07 cohort, the capture pause (resumes only per pattern doc section 10).

## 11. open issues

- **823281 has TWO active rows** (6a37433e + 1ede423e, both unsettled from 06-28) -- the
  second duplicate-active hazard (with 824662). a small run-hygiene slice should decide
  supersession/exclusion for both pairs before anyone settles those gamePks.
- the 4 legacy deliberate divergences are unsettled/excluded -- if the 823281 pair is
  ever legitimately settled it would create the first deliberate ledger entries; that
  decision belongs to the hygiene slice, not to settlement automation.
- prompt market context hardening v1 remains the prevention slice (approval-gated, paid
  canary, measured against this guard's baseline: FAIL 10 / 285).
- the 2026-07-07 cohort settles when check-settlement-finals.ps1 returns READY
  (~00:50 ET 2026-07-08); its readout must now cite the new /rows fields per pattern doc
  section 10.

## 12. recommended next slice

**(1) settle the 2026-07-07 cohort when READY** (readiness guard first; taxonomy-aware
readout consuming attributionFidelityStatus/divergenceInterpretation; 824820 will show
FAIL/AccidentalDivergence straight from /rows now). **(2) then Run Identity Hygiene v1**
(resolve the 824662 + 823281 duplicate-active pairs). **(3) then Prompt Market Context
Hardening v1** (approval-gated). capture cadence resumes only after (1) and operator
re-approval, with guard fields in every readout.

## 13. suggested next prompt

settlement: re-run the taxonomy-aware settlement resume prompt after
check-settlement-finals.ps1 returns READY, adding: "the readout's divergence-row notes
must quote the live /rows attributionFidelityStatus, attributionFidelityReason, and
divergenceInterpretation for 824820 (expected: FailMarketAttributionMismatch /
prose_claims_home_but_staged_consensus_is_away / AccidentalDivergence), and only
CountsAsCandidateEdge rows may be described as candidate edge signals (currently none)."
