---
title: "HANDOFF: Pre-Settlement QA Script + Cohort Selection Discipline v1 (2026-07-06)"
type: "handoff"
date: "2026-07-06"
status: "COMPLETE -- script shipped + validated; discipline pattern + template committed"
project: "DAI"
slice: "Pre-Settlement QA Script + Cohort Selection Discipline v1"
related:
  - "06 Execution/reports/pre-settlement-qa-script-and-cohort-discipline-2026-07-06-v1.md"
  - "06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-2026-07-06-v1.md"
---

# HANDOFF: Pre-Settlement QA Script + Cohort Selection Discipline v1 (2026-07-06)

## 1. objective

No-spend tooling + process hardening: convert the manual cohort integrity QA into the
read-only `preflight-settlement.ps1` dev script, and codify cohort selection / capture-run
discipline (pattern doc + frozen-slate template) so future paid captures are
measurement-grade by construction.

## 2. outcome

**COMPLETE.** Script shipped and validated live against the 6-run cohort in strict mode:
exit 0, found 6/6, ready 6, blockers 0, warnings 0, **agree 5 / disagree 1, divergence
= 823036** -- byte-for-byte the expected result. Negative path (bogus gamePk) exits 2.
Discipline pattern + slate template added to `06 Execution/patterns/`. Settlement-override
gate honored: finals were NOT available (all 6 games pre-game at 16:55 ET), so this tooling
slice ran legitimately.

## 3. repo state before / after

- dai before: `d79c38f`, dirty only csproj phantom, 0/0. after: `d79c38f` + 1 commit
  (`tools(sports): add pre-settlement cohort preflight`), phantom unchanged, 1 ahead.
- dai-vault before: `ab79a1a`, 4 ahead, untracked synopsis. after: `ab79a1a` + 1 docs
  commit, **5 ahead**, synopsis still untracked. Raw JSON manifest left uncommitted.

## 4. services used

devcore-sql (restarted; read-only queries for count proofs), DevCore.Api :5007 (restarted;
GET-only: /rows, /{id}, /{id}/evaluation, /reconcile-precheck). agent-service: never
started. StatsAPI: one read-only schedule call (finals gate). No paid services.

## 5. work performed

Phase 0 state + finals gate (all 6 still pre-game) -> read endpoint DTOs
(PromptRouteCalibrationRow = the primary read model incl. structured market baseline;
ReconcilePrecheckResult; AgentRunDetailDto; evaluation 404-on-absent) -> wrote the script
(read-only by construction) -> validated live: found + fixed 2 PS semantics defects
(non-enumerated IRM array -> pipeline flatten; `$run?.` variable-parse gotcha ->
`${run}?.`) -> strict validation passed exactly; negative path exits 2 -> no-writes proof
(279/112/112 unchanged after 4 script runs; only `Method Get` in script) -> wrote
discipline pattern + slate template + report + this handoff -> commits (dai script; vault
docs).

## 6. files changed

- dai (committed): `scripts/dev/sports/preflight-settlement.ps1` (new, read-only dev tool).
- dai-vault (committed):
  - `06 Execution/reports/pre-settlement-qa-script-and-cohort-discipline-2026-07-06-v1.md`
  - `06 Execution/reports/pre-settlement-qa-script-and-cohort-discipline-handoff-2026-07-06-v1.md`
  - `06 Execution/patterns/cohort-selection-and-run-discipline-v1.md`
  - `06 Execution/patterns/frozen-cohort-slate-template-v1.md`
- uncommitted by choice: `06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json`
  (regenerable script output).

## 7. db writes / side effects

**0 DB writes.** AgentRuns 279, outcomes 112, evaluations 112 -- identical before and after
all script executions. Script uses GET endpoints only; never calls /reconcile. Services
devcore-sql + DevCore.Api :5007 left RUNNING (useful for the imminent settlement slice).

## 8. paid calls / cost

0 paid model calls, $0.00. agent-service never started.

## 9. validation proof

- Strict run: exit 0; summary `{target 6, found 6, ready 6, warnings 0, blockers 0,
  outcomesPresent 0, evaluationsPresent 0, agree 5, disagree 1}`; divergence 823036 with
  lean home vs market away, gap 0.037, SingleMatch, ready=true.
- Negative run (gamePk 999999): blocker + exit 2.
- Read-only: only `Method Get` in the script (grep-verified); DB counts unchanged.
- dai diff = the new script only (+ pre-existing phantom).

## 10. what did not change

Runtime app code, prompts, registry recipes, routing, confidence generation, calibration
gate, buyer copy, schema/migrations, captured artifacts, .env: unchanged. No settlement
performed; cohort still 6/6 unreconciled.

## 11. open issues

- Cohort finals land ~00:30 ET 2026-07-07; settlement slice is the next evidence action.
- SupersededBy + per-run durable cost evidence remain read-model gaps (Durable Cost
  Evidence v1 is the queued follow-up; a supersededBy field on /rows is a candidate rider).
- dai 1 ahead + dai-vault 5 ahead, unpushed (push on approval).
- Services left running for the settlement slice; stop them if settlement is deferred.

## 12. recommended next slice

**Backed-Depth Divergence Settlement / Reconciliation v1** (after official finals): Phase 0
= `preflight-settlement.ps1` strict (expect 6/6 ready), then identity `/reconcile` x6,
outcomes + evaluations, pooled-calibration re-read. Watch 823036.

## 13. suggested next prompt

```text
SLICE: Backed-Depth Divergence Settlement / Reconciliation v1
Date: <after 2026-07-06 finals available>  |  Mode: settlement-only, no paid generation.
SETTLEMENT_APPROVED=true, GENERATION_IN_THIS_SLICE=false, PUSH_ALLOWED=false.

Settle the 6-run 2026-07-06 backed_depth divergence cohort (gamePks 822958, 822712, 824900,
823036, 823282, 823205).

Phase 0 verify: dai = d79c38f + preflight script commit (csproj phantom only); baselines
AgentRuns=279, outcomes=112, evaluations=112. Run the NEW preflight as the gate:
  .\scripts\dev\sports\preflight-settlement.ps1 -Competition mlb `
    -GamePks 822958,822712,824900,823036,823282,823205 `
    -RequireRegistry -RequireBackedDepth -RequireUnreconciled
Expect exit 0 with 6/6 ready (attach the manifest). Abort on any blocker.
Phase 1 services: devcore-sql + DevCore.Api :5007 (agent-service NOT needed; no model calls).
Phase 2 fetch official finals (StatsAPI) for all 6 gamePks; confirm all Final; record scores.
Phase 3 settle via identity POST /reconcile (all SingleMatch per preflight); write outcomes +
evaluations ONLY for these 6. Record DAI correct/incorrect per run; special attention to the
divergence run 823036 (DAI Cardinals home vs market Brewers away).
Phase 4 re-read pooled calibration (/metrics + /rows, ExclusionReason IS NULL); report whether
disagreement/coverage/discrimination moved. Do NOT change the gate (expect still FALSE).
Phase 5 re-run the preflight (expect outcomes present now -- INFO, not blocker, without
-RequireUnreconciled) as the post-settlement residue check.
Hard constraints: no generation, no prompt/routing/confidence/buyer/gate/schema change; do not
push. Vault report + continuation-grade handoff (canonical 13-section). Commit docs-only.
```

---
Durable source of truth: `pre-settlement-qa-script-and-cohort-discipline-2026-07-06-v1.md`.
This handoff is the compressed resume point.
