---
title: "HANDOFF: Diagnostic Readout / Failure Taxonomy v1 -- DONE; H1 SYSTEMIC (6/6 accidental), capture paused pending guard"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- 38-artifact audit; zero deliberate divergences; guard spec seeded"
project: "DAI"
slice: "Diagnostic Readout / Failure Taxonomy v1"
related:
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
  - "06 Execution/reports/backed-depth-prediction-failure-analysis-2026-07-07-v1.md"
  - "06 Execution/reports/backed-depth-capture-settlement-attempt-handoff-2026-07-07-v2.md"
---

# HANDOFF: diagnostic readout / failure taxonomy v1 (2026-07-07)

## 1. objective

read-only corpus audit of staged-market-vs-prose consistency across backed_depth
artifacts; split the divergence ledger deliberate/accidental/unclear; recommend whether
capture cadence stays paused; seed the guard spec.

## 2. outcome

COMPLETE. report:
`06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md`.
**H1 SYSTEMIC and broader than hypothesized: 38 artifacts audited (36 backed_depth from
5 slates + the 2 other-route settled disagreement rows). ALL SIX persisted
marketAgreement=false rows are ACCIDENTAL divergences (824500, 824743, 823529, 824662,
823036, 824820) -- including both that settled CORRECT. zero deliberate divergences
exist. the defect also hits agree rows: 824012 inverts the market against dai's own
lean, 824175 self-contradicts, no-lean canary 822882 inverts -- 9/38 (~24%) misattribute
the market in prose; 7/7 rows where staged opposed the narrative were flipped to agree
(rubber-stamp confirmed).** candidate-edge ledger: zero rows qualify; the settled 2/5
disagreement record measures a defect byproduct, not contrarian judgment (persisted
fields remain decision-time truth; gate counts untouched). **operational recommendation:
PAUSE capture cadence until Market Attribution Fidelity Guard v1 exists** (spec seeded:
PASS/FAIL_MARKET_ATTRIBUTION_MISMATCH/UNCLEAR; fails prose-claims-support-when-staged-
opposes; blocks edge interpretation, never settlement). audit byproducts: gamePk 824662
has TWO active rows (2cde423e 06-28 unsettled + 4cbd433e 06-29 settled) = precheck
hazard; 822956 invents "-1.5" spread language on an h2h-only staged market.

## 3. repo state before / after

- dai: main @ `dbda7a8`, 0/0, phantom csproj only. UNCHANGED.
- dai-vault before: `9f54c6c`, 0/0, known untracked noise. after: report + this handoff
  committed and pushed (sha in session closeout).

## 4. services used

devcore-sql + DevCore.Api :5007 read-only: /rows + 38x GET /api/agent-runs/{id}/artifact.
agent-service NOT started; no external calls; no statsapi.

## 5. work performed

skills gate (incl. dai-signal-follow-up-diagnostics discipline) -> phase 0 baselines
clean -> corpus built from live /rows (36 backed_depth by selected-or-observed regime +
2 other-route disagreement rows) -> per-artifact extraction (staged market detail +
every market sentence from summary/discern/counterCase/wwctr/lean; digest archived in
session scratchpad) -> prose-side resolution by team-name matching, manually verified on
all disagreement + ambiguous rows (incl. the 824662 duplicate-row resolution by
agentRunId) -> taxonomy defined (6 classes) -> 38-row classification table with staged +
prose evidence -> ledger split (6 divergences, all accidental, all "counts as candidate
edge: NO") -> direct answers to the 7 interpretation questions -> guard spec seed ->
pause recommendation -> report + handoff -> commit + push.

## 6. files changed

dai: none.
dai-vault (one commit):
- `06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md` (new)
- `06 Execution/reports/diagnostic-readout-failure-taxonomy-handoff-2026-07-07-v1.md`
  (this file)

## 7. db writes / side effects

0 db writes. rows 285 / settled 118 unchanged. 2026-07-07 cohort untouched (still
unsettled, 6/6 active). artifacts cached in session scratchpad only (regenerable).

## 8. paid calls / cost

0 paid model calls, $0.00.

## 9. validation proof

- every mismatch classification carries verbatim staged evidence (consensus side from
  sourceDepth market_odds detail or persisted marketConsensusSide) and verbatim prose
  quotes (in report sections 5-6); aligned rows carry the resolved prose-side vs staged
  comparison in the table; full extracted text in the session digest.
- 823036 and 824820 explicitly re-classified (both AD), consistent with the failure
  analysis and the settlement-resume handoff.
- controls: 29/31 agree rows market-aligned (the 2 exceptions classified SPM);
  6/6 disagree rows accidental.
- no accidental row is called candidate edge anywhere; gate logic untouched; no code/
  prompt/db/paid/buyer changes (dai status = phantom only; agent-service down; counts
  unchanged).

## 10. what did not change

gate-4 criterion and persisted counts, all code and prompts, confidence rules, models,
buyer copy, registry flags, database contents, the in-flight 2026-07-07 cohort.

## 11. open issues

- capture cadence PAUSED (recommendation) until Market Attribution Fidelity Guard v1 is
  live; the 07-07 in-flight cohort still settles when finals post.
- duplicate active rows for gamePk 824662 (2cde423e unsettled 06-28 + 4cbd433e settled
  06-29): reconcile-precheck would return MultipleMatches for this identity; needs a
  supersession/exclusion decision in a small hygiene slice.
- guard placement decision (compose-time vs settlement-time vs both) deferred to the
  implementation slice; corpus evidence favors both.
- market-TYPE fidelity (822956's invented spread reference) should be considered in the
  guard spec.
- readout language rules from taxonomy section 7.6 apply to ALL future readouts,
  starting with tonight's 07-07 settlement readout.

## 12. recommended next slice

sequence: (1) **settle the 2026-07-07 cohort when finals post** (~01:00 ET 07-08) --
resume prompt unchanged, readout carries the 824820 accidental annotation + taxonomy
language rules; (2) **Market Attribution Fidelity Guard v1** (code, TDD, approval-gated,
implements the section 8 spec; consider the compose-time + settlement-time split);
(3) resume capture cadence only after the guard is live.

## 13. suggested next prompt

settlement (first): re-run the resume settlement prompt after statsapi shows all 6
Final, with the phase-8 addition from
backed-depth-capture-settlement-attempt-handoff-2026-07-07-v2.md section 13, plus:
"apply diagnostic-readout-failure-taxonomy-2026-07-07-v1.md section 7.6 language rules
in the readout (market-opposed rows; taxonomy class per divergence row; candidate edge
reserved for deliberate divergences, currently zero)."

guard (second): "SLICE: Market Attribution Fidelity Guard v1. mode: code-allowed (dai),
TDD against the agent-service/.NET suites as appropriate, no paid calls, no captures, no
settlement writes, no gate/threshold/prompt changes beyond the guard itself,
approval-gated. implement the spec in diagnostic-readout-failure-taxonomy-2026-07-07-v1.md
section 8: deterministic prose-vs-staged-market fidelity check with
PASS/FAIL_MARKET_ATTRIBUTION_MISMATCH/UNCLEAR outputs; decide placement (compose-time
artifact quality warning + settlement-time divergence classification); surface the
classification additively on /rows; never block settlement by itself; block
candidate-edge ledger credit. /metrics byte-identical; full test suites green; 13-section
handoff. also resolve the 824662 duplicate-active hazard (supersession/exclusion
decision) if in scope, else name it for a hygiene slice."
