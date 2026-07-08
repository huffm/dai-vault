---
title: "Gate-4 Evidence Readout: Backed-Depth Capture Cohort 2026-07-07 (settled 2026-07-08)"
type: "report"
date: "2026-07-08"
status: "COMPLETE -- second filled instance of the gate-4 evidence readout template"
project: "DAI"
slice: "RESUME: 2026-07-07 Capture Cohort Settlement v1"
template: "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
related:
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
  - "06 Execution/reports/backed-depth-capture-cohort-2026-07-07-v1.md"
  - "06 Execution/reports/backed-depth-capture-settlement-attempt-handoff-2026-07-07-v2.md"
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
  - "06 Execution/patterns/market-attribution-fidelity-guard-v1.md"
---

# gate-4 evidence readout: backed-depth capture cohort 2026-07-07

second filled instance of the gate-4 evidence readout template v1. finals verified via
check-settlement-finals.ps1 (READY, 6/6 abstractGameState=Final codedGameState=F,
2026-07-08, same session, immediately before preflight); strict preflight-settlement.ps1
exit 0 (6/6 ready, 0 warnings, 0 blockers); before column = live /rows read pre-write;
after column = live /rows read immediately after the 6 reconcile writes. summary computed
by `pooled_calibration_summary` (agent-service `scripts/pooled_calibration_report.py`,
pure local read -- no agent-service process, no model calls). this readout applies the
taxonomy language rules (taxonomy report section 7, finding 6): persisted
marketAgreement=false rows are "market-opposed rows"; "candidate edge signal" is reserved
for deliberate divergences, of which there are currently ZERO settled-valid.

## 1. cohort header

- date settled: 2026-07-08 (~17:27 UTC)
- cohort: Backed-Depth Capture Cohort 2026-07-07 v1 (frozen slate:
  [[frozen-backed-depth-capture-slate-2026-07-07-v1]]; capture:
  [[backed-depth-capture-cohort-2026-07-07-v1]])
- runs settled this cohort: 6 (6 of 6 expected; none skipped)
- source provider: mlb_statsapi
- data regime: starter_enriched_market_backed_depth (selectedDataRegime; registry-routed,
  promptSource=registry on all 6)
- market-opposed row present: yes; gamePk 824820 (CHC@BAL) -- classified ACCIDENTAL
  divergence / market-attribution failure (see section 3)

## 2. before/after evidence table

before = live /rows read 2026-07-08 pre-write; after = live /rows read post-write.
valid denominator = settled AND exclusionReason IS NULL throughout (binding, from /rows;
/metrics not used).

| measure (field / threshold)                                  | before | after | delta |
| ------------------------------------------------------------ | ------ | ----- | ----- |
| valid settled n (outcomeStatus set, exclusionReason null)     | 116    | 122   | +6    |
| directional n (leanSide home/away)                            | 98     | 104   | +6    |
| settled rows with marketAgreement = false (market-opposed)    | 5      | 6     | +1    |
| market disagreement n (marketDisagreementN / >= 10)           | 5      | 6     | +1 (still below threshold) |
| market coverage (marketCoverage / >= 0.60)                    | 0.5918 | 0.6154 | +0.0236 (**now MEETS threshold**) |
| populated confidence regions (populatedRegionCount / >= 2)    | 2      | 2     | 0 (meets threshold) |
| region accuracy lt_0.70 (n)                                   | 0.5000 (n=6, unpopulated) | 0.5000 (n=6, unpopulated) | 0 |
| region accuracy 0.70_0.74 (n)                                 | 0.7500 (n=8, unpopulated) | 0.7500 (n=8, unpopulated) | 0 |
| region accuracy 0.75_0.79 (n)                                 | 0.5882 (n=68, populated) | 0.6027 (n=73, populated) | +0.0145 (n +5) |
| region accuracy gte_0.80 (n)                                  | 0.5000 (n=16, populated) | 0.4706 (n=17, populated) | -0.0294 (n +1) |
| discrimination status + top-bottom delta                      | inverted; top gte_0.80 vs bottom 0.75_0.79; delta -0.0882 | inverted; top gte_0.80 vs bottom 0.75_0.79; delta -0.1321 | -0.0439 (inversion deeper) |
| failingReasons (full list)                                    | discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage | discrimination_inverted, insufficient_market_disagreement | **insufficient_market_coverage RESOLVED** |
| conclusionsAllowed                                            | false  | false | unchanged |

legacy exact-2dp bucket accuracies (byConfidenceBucket), informational continuity only,
never a verdict input: 0.75 n 68->73 acc 0.5882->0.6027; 0.80 n 16->17 acc
0.5000->0.4706; 0.63/0.68/0.70/0.72 unchanged (n 1/5/6/2).

## 3. per-run outcome list

| agentRunId | gamePk | matchup (away@home) | dai lean | market side | agree | final | evaluation | market-opposed | provenance |
| ---------- | ------ | ------------------- | -------- | ----------- | ----- | ----- | ---------- | -------------- | ---------- |
| 9e2c433e-f36b-1410-8178-00373db4b724 | 823687 | CLE @ MIN  | home | home (MIN) | yes | home_win (CLE 1 - MIN 3)  | correct   | no  | statsapi / gamePk 823687 final away 1 home 3 / notes complete |
| a32c433e-f36b-1410-8178-00373db4b724 | 824820 | CHC @ BAL  | home | away (CHC) | NO  | away_win (CHC 5 - BAL 2)  | incorrect | **yes** | statsapi / gamePk 824820 final away 5 home 2 / notes complete |
| a92c433e-f36b-1410-8178-00373db4b724 | 822956 | NYY @ TB   | home | home (TB)  | yes | home_win (NYY 4 - TB 6)   | correct   | no  | statsapi / gamePk 822956 final away 4 home 6 / notes complete |
| aa2c433e-f36b-1410-8178-00373db4b724 | 822713 | HOU @ WSH  | home | home (WSH) | yes | away_win (HOU 6 - WSH 3)  | incorrect | no  | statsapi / gamePk 822713 final away 6 home 3 / notes complete |
| ac2c433e-f36b-1410-8178-00373db4b724 | 823280 | ARI @ SD   | home | home (SD)  | yes | home_win (ARI 1 - SD 4)   | correct   | no  | statsapi / gamePk 823280 final away 1 home 4 / notes complete |
| b32c433e-f36b-1410-8178-00373db4b724 | 824579 | BOS @ CWS  | away | away (BOS) | yes | away_win (BOS 8 - CWS 1)  | correct   | no  | statsapi / gamePk 824579 final away 8 home 1 / notes complete |

- cohort record: 4 correct / 2 incorrect (0.6667). all six settled via identity
  POST /api/agent-runs/reconcile, SingleMatch on every write; settlementSource,
  settlementSourceRef, settlementNotes non-null on all six (residue contract satisfied).
- the lone gte_0.80 run this cohort (822713 HOU@WSH, conf 0.80, agree) settled
  incorrect -- this is what deepened the discrimination inversion.
- **market-opposed row 824820 (CHC@BAL): divergence = ACCIDENTAL (H1) --
  market-attribution failure, NOT a candidate edge signal.** artifact prose asserted
  market support for the Orioles; staged consensus was away Cubs (see
  [[backed-depth-capture-settlement-attempt-handoff-2026-07-07-v2]] section 2). live
  /rows guard fields at settlement: attributionFidelityStatus =
  `FailMarketAttributionMismatch`; attributionFidelityReason =
  `prose_claims_home_but_staged_consensus_is_away`; divergenceInterpretation =
  `AccidentalDivergence`. per the CountsAsCandidateEdge ledger rule
  ([[market-attribution-fidelity-guard-v1]]), only deliberate divergences count as
  candidate edge signals; this row does not count and does not strengthen the edge
  narrative. dai leaned home (baltimore-orioles, 0.75); market consensus away
  (chicago-cubs, gap 0.03); final CHC 5 - BAL 2: dai incorrect, market side correct.
- settled market-opposed ledger after this cohort: marketAgreement=false n=6,
  2 correct / 4 incorrect (0.3333). n=6 is below minMarketDisagreementReadableN=10 and
  is not readable. per the taxonomy, every settled-valid entry in this ledger is an
  accidental divergence; the deliberate candidate-edge ledger remains EMPTY (zero
  settled-valid CountsAsCandidateEdge rows).
- marketDisagreementN=6 is one short of the n=7 re-projection checkpoint from
  [[evidence-acquisition-cadence-proposal-2026-07-07-v1]]; re-projection is NOT yet
  triggered.

## 4. gate-4 verdict

gate 4: FALSE unless the live criterion says otherwise; a TRUE here requires merit
verification, not celebration.

recorded verbatim from the live post-settlement payload (discrimination_hybrid_v1):

- conclusionsAllowed = false
- failingReasons = ["discrimination_inverted", "insufficient_market_disagreement"]

conclusionsAllowed is false; no sub-gate true-verification is triggered. this cohort
resolved one of the three prior failures: market coverage crossed its threshold
(0.5918 -> 0.6154 vs 0.60; the projection's "+2 covered rows" gap was met with +6).
disagreement n moved +1 to 6 (needs 10). the discrimination inversion deepened
(delta -0.0882 -> -0.1321) because the single 0.80-confidence run settled incorrect;
the gte_0.80 region (n=17, 0.4706) still sits below 0.75_0.79 (n=73, 0.6027). the
evidence-sufficiency projection's warning stands: discrimination is not
volume-purchasable and a true inversion will not settle out by adding rows.

## 5. what this readout does not license

- no tuning
- no threshold edits
- no model replacement
- no buyer-facing accuracy, edge, or performance claims
- no gate edits
- no registry default-on change
- no capture resumption (separate re-approval required; the capture pause interacts with
  the fidelity-guard baseline and Prompt Market Context Hardening v1 sequencing)
