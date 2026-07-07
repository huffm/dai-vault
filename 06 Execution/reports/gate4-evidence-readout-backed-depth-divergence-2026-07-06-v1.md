---
title: "Gate-4 Evidence Readout: Backed-Depth Divergence Cohort 2026-07-06 (settled 2026-07-07)"
type: "report"
date: "2026-07-07"
status: "COMPLETE -- first filled instance of the gate-4 evidence readout template"
project: "DAI"
slice: "Backed-Depth Divergence Settlement / Reconciliation v1"
template: "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
related:
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
  - "06 Execution/reports/backed-depth-divergence-candidate-slate-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-settlement-attempt-handoff-2026-07-06-v1.md"
---

# gate-4 evidence readout: backed-depth divergence cohort 2026-07-06

first filled instance of the gate-4 evidence readout template v1. before column re-verified
fresh against a live /rows read at settlement time (2026-07-07), not taken from the blocked
handoff's preserved snapshot; after column read from live /rows immediately after the 6
reconcile writes. summary computed by `pooled_calibration_summary`
(agent-service `scripts/pooled_calibration_report.py`, pure local read -- no agent-service
process, no model calls).

## 1. cohort header

- date settled: 2026-07-07
- cohort: Backed-Depth Divergence Cohort 2026-07-06 v2 (frozen slate:
  [[backed-depth-divergence-candidate-slate-2026-07-06-v2]]; capture:
  [[backed-depth-divergence-capture-2026-07-06-v2]])
- runs settled this cohort: 6 (6 of 6 expected; none skipped)
- source provider: mlb_statsapi
- data regime: starter_enriched_market_backed_depth (selectedDataRegime; registry-routed,
  promptSource=registry on all 6)
- backed_depth divergence present: yes; divergent gamePk 823036 (MIL@STL)

## 2. before/after evidence table

before = live /rows read 2026-07-07 pre-write; after = live /rows read post-write.
valid denominator = settled AND exclusionReason IS NULL throughout (binding, from /rows;
/metrics not used).

| measure (field / threshold)                                  | before | after | delta |
| ------------------------------------------------------------ | ------ | ----- | ----- |
| valid settled n (outcomeStatus set, exclusionReason null)     | 110    | 116   | +6    |
| directional n (leanSide home/away)                            | 92     | 98    | +6    |
| settled rows with marketAgreement = false                     | 4      | 5     | +1    |
| market disagreement n (marketDisagreementN / >= 10)           | 4      | 5     | +1 (still below threshold) |
| market coverage (marketCoverage / >= 0.60)                    | 0.5652 | 0.5918 | +0.0266 (still below threshold) |
| populated confidence regions (populatedRegionCount / >= 2)    | 2      | 2     | 0 (meets threshold) |
| region accuracy lt_0.70 (n)                                   | 0.5000 (n=6, unpopulated) | 0.5000 (n=6, unpopulated) | 0 |
| region accuracy 0.70_0.74 (n)                                 | 0.7500 (n=8, unpopulated) | 0.7500 (n=8, unpopulated) | 0 |
| region accuracy 0.75_0.79 (n)                                 | 0.6190 (n=63, populated) | 0.5882 (n=68, populated) | -0.0308 (n +5) |
| region accuracy gte_0.80 (n)                                  | 0.5333 (n=15, populated) | 0.5000 (n=16, populated) | -0.0333 (n +1) |
| discrimination status + top-bottom delta                      | inverted; top gte_0.80 vs bottom 0.75_0.79; delta -0.0857 | inverted; top gte_0.80 vs bottom 0.75_0.79; delta -0.0882 | -0.0025 (inversion slightly deeper) |
| failingReasons (full list)                                    | discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage | discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage | unchanged |
| conclusionsAllowed                                            | false  | false | unchanged |

legacy exact-2dp bucket accuracies (byConfidenceBucket), informational continuity only,
never a verdict input: 0.75 n 63->68 acc 0.6190->0.5882; 0.80 n 15->16 acc 0.5333->0.5000;
0.63/0.68/0.70/0.72 unchanged (n 1/5/6/2).

## 3. per-run outcome list

| agentRunId | gamePk | matchup (away@home) | dai lean | market side | agree | final | evaluation | divergence | provenance |
| ---------- | ------ | ------------------- | -------- | ----------- | ----- | ----- | ---------- | ---------- | ---------- |
| ac31433e-f36b-1410-8175-00373db4b724 | 822958 | NYY @ TB  | home | home (TB)  | yes | away_win (NYY 5 - TB 1)   | incorrect | no  | statsapi / gamePk 822958; final NYY 5 @ TB 1 / notes complete |
| ad31433e-f36b-1410-8175-00373db4b724 | 822712 | HOU @ WSH | home | home (WSH) | yes | home_win (HOU 11 - WSH 12) | correct   | no  | statsapi / gamePk 822712; final HOU 11 @ WSH 12 / notes complete |
| b331433e-f36b-1410-8175-00373db4b724 | 824900 | NYM @ ATL | home | home (ATL) | yes | away_win (NYM 7 - ATL 6)  | incorrect | no  | statsapi / gamePk 824900; final NYM 7 @ ATL 6 / notes complete |
| b431433e-f36b-1410-8175-00373db4b724 | 823036 | MIL @ STL | home | away (MIL) | NO  | away_win (MIL 4 - STL 3)  | incorrect | **yes** | statsapi / gamePk 823036; final MIL 4 @ STL 3 / notes complete |
| b631433e-f36b-1410-8175-00373db4b724 | 823282 | ARI @ SD  | home | home (SD)  | yes | away_win (ARI 8 - SD 0)   | incorrect | no  | statsapi / gamePk 823282; final ARI 8 @ SD 0 / notes complete |
| b731433e-f36b-1410-8175-00373db4b724 | 823205 | TOR @ SF  | away | away (TOR) | yes | home_win (TOR 1 - SF 10)  | incorrect | no  | statsapi / gamePk 823205; final TOR 1 @ SF 10 / notes complete |

- cohort record: 1 correct / 5 incorrect (0.1667). all six settled via identity
  POST /api/agent-runs/reconcile, SingleMatch on every write; settlementSource,
  settlementSourceRef, settlementNotes non-null on all six (residue contract satisfied).
- finals verified fresh against statsapi schedule (sportId=1, date=2026-07-06) at
  settlement time: all six abstractGameState=Final, codedGameState=F. three games flipped
  relative to the in-progress scores in the blocked handoff (824900, 823036, 822712),
  which is exactly why the finals gate re-verification is mandatory.
- divergence row 823036 (MIL@STL): the cohort's only marketAgreement=false row and the
  first settled backed_depth divergence. dai leaned home (st-louis-cardinals, 0.75);
  market consensus was away (milwaukee-brewers, favP 0.521, gap 0.042). final MIL 4 - STL 3:
  dai incorrect, market side correct. this is a candidate edge signal observation only --
  a single settled divergence licenses nothing in either direction.
- pooled settled-divergence ledger after this cohort: marketAgreement=false n=5,
  2 correct / 3 incorrect (0.40). candidate edge signal only; n=5 is half the
  minMarketDisagreementReadableN=10 threshold and is not readable.

## 4. gate-4 verdict

gate 4: FALSE unless the live criterion says otherwise; a TRUE here requires merit
verification, not celebration.

recorded verbatim from the live post-settlement payload (discrimination_hybrid_v1):

- conclusionsAllowed = false
- failingReasons = ["discrimination_inverted", "insufficient_market_disagreement",
  "insufficient_market_coverage"]

conclusionsAllowed is false; no sub-gate true-verification is triggered. the three failures
after settlement are the same three as before; this cohort moved coverage (+0.0266 to
0.5918) and disagreement n (+1 to 5) toward their thresholds while deepening the
discrimination inversion slightly (delta -0.0857 -> -0.0882: the lone 0.80-confidence run
settled incorrect).

## 5. what this readout does not license

- no tuning
- no threshold edits
- no model replacement
- no buyer-facing accuracy, edge, or performance claims
- no gate edits
- no registry default-on change
