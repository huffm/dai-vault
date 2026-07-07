---
title: "HANDOFF: Backed-Depth Prediction Failure Analysis v1 -- DONE (2026-07-07)"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- diagnostic report shipped; key finding: the first divergence was accidental (market misattribution in prose)"
project: "DAI"
slice: "Backed-Depth Prediction Failure Analysis v1"
related:
  - "06 Execution/reports/backed-depth-prediction-failure-analysis-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md"
---

# HANDOFF: backed-depth prediction failure analysis v1 (2026-07-07)

## 1. objective

read-only diagnosis of the settled 2026-07-06 backed_depth cohort (1 correct / 5
incorrect, divergence 823036 incorrect): failure modes, per-run classification, ranked
root-cause hypotheses, improvement backlog -- no changes implemented.

## 2. outcome

COMPLETE. report:
`06 Execution/reports/backed-depth-prediction-failure-analysis-2026-07-07-v1.md`.
**headline finding (H1, confirmed from artifacts): the "first backed_depth divergence"
was an ACCIDENT -- 823036's staged market data said `consensus away, median home win
prob 50%`, but the artifact's summary AND discern phase claim the market preferred the
Cardinals (home), and the counter-case misplaces the away starter at home. the model
believed the market agreed with it; the divergence row measures a misread, not an edge
attempt.** the direction-integrity guard correctly returned Consistent (it checks
lean-vs-prose only) -- prose-vs-staged-market is an unguarded consistency class. other
findings: the remaining five runs are market echo (lean == consensus, consensus cited as
support; market went 1-4 on those games; P(<=1 win) ~ 0.15 = mostly variance); confidence
is template-driven (uniform 0.75 default, 0.80 for signal coherence -- the 0.80 lost;
aConf==conf, evR frozen at 2, identical evidence profile on the win and the losses).
NO tuning proposed; backlog split allowed-now (read-only market-attribution corpus
audit, failure taxonomy in readouts, divergence post-mortems) / candidate-later
(market-attribution guard, prompt hardening, confidence review -- all gated) /
forbidden-now.

## 3. repo state before / after

- dai: main @ `dbda7a8`, 0/0, phantom csproj only. UNCHANGED.
- dai-vault before: main @ `b1b4490`, 0/0, known untracked noise. after: report + this
  handoff committed and pushed (sha in session closeout); noise untouched.

## 4. services used

devcore-sql + DevCore.Api :5007, read-only: GET /api/agent-runs/{id}/artifact x6 (the
2026-07-06 cohort's OutputJson), /rows context from earlier same-session reads.
agent-service NOT started; no external calls; no statsapi needed.

## 5. work performed

- skills gate; dai-signal-follow-up-diagnostics loaded per its trigger (artifact-level
  diagnosis) and its discipline applied (canonical signal vocabulary, phase ownership,
  fallback-ladder framing, no invented staleness).
- phase 0 baselines clean (dai `dbda7a8`, vault `b1b4490`).
- phase 1: all six artifacts pulled to scratchpad; reconstruction table built from
  artifact + /rows + finals (settlement handoff).
- phase 2: pattern analysis across side bias, market relationship, confidence, evidence
  profile (identical on all six -- the win is indistinguishable from the losses),
  feature weighting (season starter aggregates + consensus echo; nothing else staged),
  prose-lean consistency (6/6 Consistent), and the NEW market-attribution class (5/6
  match staged consensus; 823036 does not).
- phase 3: six ranked hypotheses (H1 misattribution CONFIRMED; H2 market echo; H3
  confidence template; H4 variance; H5 thin+understated signal vocabulary
  [missingSignals=[] while discern prose says lineup data missing]; H6 shallow starter
  comparison), each with confirm/falsify/testability/lock status.
- phases 4-5: three-tier improvement backlog; report written with all required sections
  and the fixed does-not-license list.
- phases 6-8: validation, vault-only commit, push.

## 6. files changed

dai: none.
dai-vault (one commit):
- `06 Execution/reports/backed-depth-prediction-failure-analysis-2026-07-07-v1.md` (new)
- `06 Execution/reports/backed-depth-prediction-failure-analysis-handoff-2026-07-07-v1.md`
  (this file)

## 7. db writes / side effects

0 db writes (artifact GETs only). the 2026-07-07 capture cohort was NOT touched (noted
as in-flight only). services left as found.

## 8. paid calls / cost

0 paid model calls, $0.00.

## 9. validation proof

- every claim traceable: artifact JSONs in session scratchpad (staged sourceDepth market
  detail vs prose quotes), /rows fields, settlement handoff finals, readout numbers.
- 823036 evidence pair quoted verbatim in the report: staged "consensus away, median
  home win prob 50%" vs summary/discern "market ... preference for the Cardinals."
- no /metrics used; no code/prompt/gate/threshold/buyer changes (dai status = phantom
  only); no paid calls (agent-service down throughout); n=6 caveats present throughout
  (H4 explicitly credits variance; no tuning recommended anywhere).

## 10. what did not change

code, prompts, confidence rules, gates/criterion, model, buyer copy, registry flags,
database contents, the unsettled 2026-07-07 cohort.

## 11. open issues

- H1 corpus scope unknown: 1 confirmed instance in 6 artifacts; the corpus-wide
  market-attribution audit (read-only) would establish the rate across all settled rows.
- the candidate-edge ledger counts 823036 as a divergence row (it legitimately is, by
  persisted fields) but it was not a deliberate contrarian read -- future readouts should
  annotate this distinction; decision belongs to the taxonomy slice.
- the 2026-07-07 cohort settles tonight/tomorrow; its divergence run 824820 CHC@BAL must
  get the H1 staged-vs-prose check during settlement.
- global home-lean underperformance (0.557 n=70 vs 0.643 n=28) remains a watch item, not
  attributable from this cohort.

## 12. recommended next slice

**Diagnostic Readout / Failure Taxonomy v1** (read-only): corpus-wide market-attribution
audit + standardize the per-run failure classification as a settlement-readout appendix
+ decide the divergence-ledger annotation for accidental divergences. HOWEVER, if the
2026-07-07 cohort is Final first, settle it before anything else (blocked-attempt
handoff governs) -- and apply the H1 check to 824820 in that settlement.

## 13. suggested next prompt

"SLICE: Diagnostic Readout / Failure Taxonomy v1. mode: read-only, docs-only, no-spend.
read first: backed-depth-prediction-failure-analysis-2026-07-07-v1.md (H1 + section 7
allowed-now backlog). objective: (1) corpus-wide market-attribution audit -- for every
settled directional row with a market side, compare the artifact's prose market claim
(summary/discern/counterCase) to persisted marketConsensusSide; tabulate matches/
mismatches/no-claim; (2) define the standard per-run failure-classification appendix for
settlement readouts; (3) propose (docs-only) the divergence-ledger annotation rule for
accidental vs deliberate divergence. deliverable: report + handoff in 06 Execution/
reports/, commit, push after verification. no code, no guard implementation (that is a
later approved slice), no tuning language."
