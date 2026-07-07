---
title: "HANDOFF: Backed-Depth Divergence Settlement v1 -- BLOCKED at finals gate (2026-07-06)"
type: "handoff"
date: "2026-07-06"
status: "BLOCKED -- finals gate; 0 settlement writes; resume post-finals"
project: "DAI"
slice: "Backed-Depth Divergence Settlement / Reconciliation v1 + First Gate-4 Evidence Readout"
related:
  - "06 Execution/reports/pre-settlement-qa-script-and-cohort-discipline-handoff-2026-07-06-v1.md"
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
---

# HANDOFF: backed-depth divergence settlement v1 -- BLOCKED at finals gate (2026-07-06)

## 1. objective

settle the 2026-07-06 backed_depth divergence cohort (6 runs) through the identity
/reconcile path after authoritative finals, then produce the first filled instance of the
gate-4 evidence readout template. secondary: harden template divergence wording to
"candidate edge signal".

## 2. outcome

BLOCKED, correctly, at the finals gate. statsapi at 2026-07-06 20:48 ET: 0 of 6 games
Final -- 822958/822712/824900/823036 In Progress, 823282/823205 Pre-Game. hard constraint
"do not reconcile pre-finals" applied; ZERO settlement writes. completed before the gate:
template wording hardening (committed), strict preflight PASS (exit 0, 6/6 ready, 0
blockers, agree 5 / disagree 1, divergence 823036), and the full binding before snapshot
(section 9). the filled readout was deliberately NOT created: a before-only readout in
reports/ could be mistaken for a completed one. the before values below are the ready-to-
paste before column for the resume run.

## 3. repo state before / after

- dai: main @ `dbda7a8` (pushed, 0 ahead), dirty = csproj phantom only. UNCHANGED; the
  preflight manifest byproduct written to repo root was deleted (regenerable).
- dai-vault before: `399f1e1`, 2 ahead. after: `ac9583b` (wording) + this handoff commit,
  4 ahead. untracked unchanged (manifest json, synopsis).

## 4. services used

docker desktop started (was down); devcore-sql container started; DevCore.Api :5007
started via scripts/start-platform-api.ps1 (GET-only usage). agent-service NEVER started
(pooled summary computed via scripts/pooled_calibration_report.py --file, pure local
python). statsapi: one read-only schedule call (finals gate). services left RUNNING for
the resume window; if the session hosting the api exits, restart via the same script.

## 5. work performed

phase 0 baselines (279 rows / 112 settled / 110 valid-settled) -> phase 1 wording
hardening committed `ac9583b` -> phase 2 strict preflight PASS (exact expected result,
byte-identical to the validated qa run) -> phase 3 before snapshot saved (rows-before.json
+ pooled-before.json in session scratchpad; binding values recorded in section 9) ->
phase 4 finals gate BLOCKED (0/6 final) -> no writes -> cleanup + this handoff.

## 6. files changed

dai-vault only:
- `06 Execution/patterns/gate4-evidence-readout-template-v1.md` (1 line: divergence rows
  are "the only candidate edge-over-market signal and do not imply demonstrated edge by
  themselves") -- committed `ac9583b`.
- this handoff (committed).
dai: none (byproduct deleted).

## 7. db writes / side effects

0 db writes. rows 279 / settled 112 identical before and after; /reconcile was never
called (the only write-capable endpoint in scope). side effects: docker desktop +
devcore-sql + DevCore.Api now running.

## 8. paid calls / cost

0 paid model calls, $0.00. agent-service never started; statsapi is free.

## 9. validation proof + preserved before snapshot (binding, exclusionReason IS NULL)

preflight: exit 0; `{target 6, found 6, ready 6, warnings 0, blockers 0, agree 5,
disagree 1}`; divergence 823036 lean home (st-louis-cardinals) vs market away
(milwaukee-brewers); all SingleMatch.

finals gate (statsapi 20:48 ET): 822958 I (nyy 5 - tb 1), 822712 I (hou 7 - wsh 12),
824900 I (nym 1 - atl 3), 823036 I (mil 0 - stl 2), 823282 P, 823205 P.

before snapshot (pooled_calibration_summary on live /rows, exclusion-filtered):
- counts: reconciled 110, directional 92, noDecision 18, settledSlates 12, excluded 16
- validDirectionalN: 92
- marketAgreement: withMarketSide 52; agreement n 48 (acc 0.6458); disagreement n 4
  (2 correct / 2 incorrect, acc 0.5000)
- marketCoverage: 0.5652 (< 0.60) | populatedRegionCount: 2
- regions: lt_0.70 n6 acc 0.5000 (unpop); 0.70_0.74 n8 acc 0.7500 (unpop);
  0.75_0.79 n63 acc 0.6190 (pop); gte_0.80 n15 acc 0.5333 (pop)
- discrimination: status inverted; top gte_0.80 vs bottom 0.75_0.79; delta -0.0857
- failingReasons: [discrimination_inverted, insufficient_market_disagreement,
  insufficient_market_coverage]
- conclusionsAllowed: false

## 10. what did not change

no settlement writes, no outcomes/evaluations, no gate/threshold/criterion change, no
pooled_calibration.py change, no prompts/routing/confidence/model/buyer copy, no registry
flag enabled, no captures, no schema change, nothing pushed. cohort remains 6/6
unreconciled.

## 11. open issues

- finals expected ~00:30 ET 2026-07-07 (west-coast games latest). resume then.
- filled gate-4 readout pending settlement (before column ready in section 9).
- services left running; may need restart if the hosting session ends.
- dai-vault 4 ahead unpushed (push on approval).

## 12. recommended next slice

resume THIS slice after all 6 games are Final: re-run strict preflight (expect unchanged
6/6 ready), re-take the before snapshot fresh (cheap; guards against any interim writes),
then phases 4-8: statsapi finals with scores, identity /reconcile x6 with full provenance,
post-settlement verification, first filled readout at
06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md,
docs-only commits, no push. do NOT start gate-4 evidence-sufficiency projection v1 until
that readout exists.

## 13. suggested next prompt

resume the settlement slice prompt from
pre-settlement-qa-script-and-cohort-discipline-handoff-2026-07-06-v1.md section 13
verbatim (with the phase-4 readout addition), noting: template wording hardening already
done (`ac9583b`); verify all 6 statsapi states are Final before any write; before-column
values may be taken from section 9 of this handoff but MUST be re-verified against a
fresh /rows read; settlement provenance source=statsapi, sourceRef=gamePk + final score,
notes=structured capture-mode string per the residue contract.
