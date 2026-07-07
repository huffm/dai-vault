---
title: "HANDOFF: Backed-Depth Divergence Settlement v1 -- SETTLED 6/6, first filled gate-4 readout (2026-07-07)"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- 6/6 reconciled, readout filled, gate 4 still FALSE on merits"
project: "DAI"
slice: "Backed-Depth Divergence Settlement / Reconciliation v1"
related:
  - "06 Execution/reports/backed-depth-divergence-settlement-attempt-handoff-2026-07-06-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md"
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
---

# HANDOFF: backed-depth divergence settlement v1 -- SETTLED (2026-07-07)

## 1. objective

settle the 2026-07-06 backed_depth divergence cohort (6 runs) through the identity
/reconcile path after fresh statsapi finals verification, and produce the first filled
gate-4 evidence readout from the template. resume of the 2026-07-06 slice that blocked
correctly at the finals gate.

## 2. outcome

COMPLETE. all 6 cohort runs reconciled cleanly (SingleMatch on every write, full
provenance, zero refusals, zero retries). cohort record 1 correct / 5 incorrect; the
divergence run 823036 (MIL@STL, dai home vs market away) settled INCORRECT -- the market
side won 4-3. first filled gate-4 readout created at
`06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md`.
gate 4 after settlement: conclusionsAllowed FALSE, failingReasons unchanged
[discrimination_inverted, insufficient_market_disagreement, insufficient_market_coverage].
coverage 0.5652 -> 0.5918 and disagreement n 4 -> 5 moved toward thresholds; the
discrimination inversion deepened slightly (-0.0857 -> -0.0882).

## 3. repo state before / after

- dai before and after: main @ `dbda7a8`, 0 ahead / 0 behind origin/main, dirty only on
  the pre-existing `DevCore.Data.csproj` phantom. NO dai changes, NO dai commit.
- dai-vault before: main @ `a26ea13`, 0 ahead / 0 behind (the 07-06 backlog was already
  pushed; the "2/4 ahead" notes in older handoffs and session memory are stale). untracked:
  `preflight-settlement-manifest-2026-07-06-v1.json`, `system-state-synopsis-v1.md` (known
  noise, intentionally not committed).
- dai-vault after: readout + this handoff committed on main; pushed (sha in section 9 /
  final summary). same two untracked files remain.

## 4. services used

- docker desktop: started (was down). devcore-sql container: started (was stopped).
- DevCore.Api :5007: started via `scripts/start-platform-api.ps1` (background). used for
  GET /rows, GET /{id}, GET /{id}/evaluation, GET reconcile-precheck (via preflight
  script), and the only write path POST /api/agent-runs/reconcile x6.
- agent-service: NEVER started (port 8000 confirmed not listening). pooled summary
  computed via `services/agent-service/scripts/pooled_calibration_report.py` (pure local
  python import of pooled_calibration, read-only).
- statsapi: free read-only schedule call for the finals gate.
- services left RUNNING (docker, devcore-sql, DevCore.Api) for any follow-up slice.

## 5. work performed

- phase 0: baselines recorded (dai `dbda7a8` clean-except-phantom; vault `a26ea13` in
  sync); docker + sql + api started; /rows reachable, baseline 279 rows / 112 settled /
  110 valid-settled (matches blocked handoff exactly).
- phase 1: cohort identity confirmed from the frozen slate/capture docs: 6 runs, prefixes
  ac31433e/ad31433e/b331433e/b431433e/b631433e/b731433e, gamePks
  822958/822712/824900/823036/823282/823205, all mlb_statsapi + backed_depth regime +
  unreconciled + exclusionReason null, divergence 823036 present.
- phase 2: finals gate PASSED fresh: statsapi schedule 2026-07-06, all 6
  abstractGameState=Final / codedGameState=F with scores (readout section 3). three games
  flipped vs the blocked handoff's in-progress scores (822712, 824900, 823036) --
  re-verification was load-bearing.
- phase 3: strict preflight PASS: exit 0, 6/6 found+ready, 0 warnings, 0 blockers,
  agree 5 / disagree 1, all SingleMatch, with -RequireRegistry -RequireBackedDepth
  -RequireUnreconciled -FailOnWarnings -ExpectedRunPrefixes. manifest saved to session
  scratchpad (regenerable byproduct; deliberately not added to either repo).
- phase 4: fresh binding before snapshot from live /rows via pooled_calibration_report
  (values in readout section 2; matches the blocked handoff's preserved snapshot on every
  measure except its `settledSlates: 12`, which the fresh read shows as 11 -- see open
  issues).
- phase 5: 6 identity POST /reconcile writes, each after a per-run re-read (identity via
  /rows row match, detail status completed, evaluation 404). request shape per
  `ReconcileOutcomeRequest`: sourceProvider mlb_statsapi, externalGameId gamePk,
  outcomeStatus home_win/away_win, scores, source `statsapi`, sourceRef
  `gamePk <pk>; final <AWY s @ HOM s>`, notes cohort + score + verification + `via
  reconcile` (+ divergence marker on 823036). all 6 returned SingleMatch with the expected
  evaluatedRunId and evalStatus.
- phase 6: post-write verification per run: outcome present, evaluation present
  (1 correct / 5 incorrect), resultSide + matchedOutcome correct, provenance triple
  non-null, exclusionReason still null. totals 279 rows / 118 settled / 116 valid-settled
  (+6 exactly); no other row changed (excluded count 16 stable, noDecision 18 stable).
- phase 7: first filled gate-4 evidence readout written (template followed verbatim,
  table filled, verdict language fixed, does-not-license list included).
- phase 8: validation (section 9).
- phases 9-10: vault-only commit + push.

## 6. files changed

dai: none.
dai-vault (committed):
- `06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md` (new)
- `06 Execution/reports/backed-depth-divergence-settlement-handoff-2026-07-07-v1.md` (this file)

## 7. db writes / side effects

- 6 AgentRunOutcome rows + 6 AgentRunEvaluation rows written via POST /reconcile
  (112 -> 118 each). no direct SQL. no other tables touched. AgentRuns count unchanged
  at 279.
- side effects: docker desktop, devcore-sql, DevCore.Api left running.

## 8. paid calls / cost

0 paid model calls, $0.00. agent-service never started; statsapi schedule reads are free;
pooled summaries computed locally from /rows json.

## 9. validation proof

- preflight exit 0: `target 6 | found 6 | ready 6 | warnings 0 | blockers 0 | agree 5 |
  disagree 1`.
- finals: all 6 statsapi Final with scores (readout section 3).
- per-run post-write re-reads: 6/6 outcome+evaluation present, provenance triple non-null,
  evals = expected from lean vs final on every run.
- /rows before 110 valid-settled -> after 116 (+6); marketAgreement=false settled rows
  4 -> 5; live discriminationHybrid after: conclusionsAllowed false, failingReasons
  [discrimination_inverted, insufficient_market_disagreement,
  insufficient_market_coverage] (recorded verbatim in the readout).
- no code changed -> no test run needed (dai git status: phantom csproj only; reconcile
  behavior proven live by 6 successful guarded writes). registry canary absent from
  agent-service `.env` and shell; port 8000 not listening; no buyer copy touched; no
  prompt/routing/confidence/gate/model file touched.

## 10. what did not change

dai code, prompts, routing, confidence logic, gate thresholds, discrimination_hybrid_v1,
pooled_calibration.py, buyer-facing copy, registry default-off posture, schema. /metrics
untouched and unused as a denominator. no captures, no new runs, no exclusions written.

## 11. open issues

- snapshot discrepancy (minor, resolved in favor of fresh data): the blocked handoff's
  preserved before snapshot recorded `settledSlates: 12`; the fresh pre-write read showed
  11 with an 11-entry bySlate table summing to 92 directional -- every other value matched
  exactly. the 12 appears to be a transcription slip; binding values were taken from the
  fresh read per the re-verify rule. after settlement, settledSlates = 12 (the 2026-07-06
  slate joined).
- backed_depth observed-route accuracy after this cohort is 6/13 (0.4615) and the settled
  divergence ledger is 2/5 -- small-n noise, no conclusions licensed, but worth carrying
  into the evidence-sufficiency projection.
- services left running; stop them if the machine needs to be quiet.
- session memory note `project-sports-calibration-state-2026-07-01` is stale on vault
  push state (says 2 ahead unpushed) and on this cohort being unsettled.

## 12. recommended next slice

Gate-4 Evidence-Sufficiency Projection v1 -- its preconditions are now met: all 6 cohort
runs reconciled cleanly, the first filled gate-4 readout exists, and the /rows after-state
is normalized and trustworthy (116 valid-settled, provenance complete). the projection
should quantify, per failing sub-gate, how much more settled evidence (and of what shape:
divergence rows for disagreement n >= 10, market-sided rows for coverage >= 0.60, and what
discrimination would need to look like) separates the current state from a merit-based
gate-4 TRUE -- no tuning, no threshold edits, read-only.

## 13. suggested next prompt

"SLICE: Gate-4 Evidence-Sufficiency Projection v1. mode: local, read-only, no paid calls,
no settlement writes, no gate/threshold/criterion edits, docs-only output. read first:
06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md,
06 Execution/reports/backed-depth-divergence-settlement-handoff-2026-07-07-v1.md, and
agent-service pooled_calibration.py (discrimination_hybrid_v1 thresholds). using the live
/rows state (116 valid-settled), project for each failingReason the minimum additional
settled evidence required to clear it (marketDisagreementN >= 10, marketCoverage >= 0.60,
discrimination non-inverted within tolerance 0.05), including which capture shapes
(divergence-priority backed_depth cohorts) contribute to which sub-gate, and the expected
number of morning capture slices at observed divergence yield (~1 per 6-run cohort).
deliverable: a projection report in 06 Execution/reports/ + updated readout appendix if
needed + 13-section handoff. do not call divergence demonstrated edge; candidate edge
signal only."
