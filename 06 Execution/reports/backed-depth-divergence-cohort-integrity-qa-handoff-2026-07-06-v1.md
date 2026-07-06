---
title: "HANDOFF: Backed-Depth Divergence Cohort Integrity QA v1 (2026-07-06)"
type: "handoff"
date: "2026-07-06"
status: "COMPLETE -- QA passed; cohort settlement-ready; no blockers"
project: "DAI"
slice: "Backed-Depth Divergence Cohort Integrity QA v1"
related:
  - "06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-2026-07-06-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
---

# HANDOFF: Backed-Depth Divergence Cohort Integrity QA v1 (2026-07-06)

## 1. objective

No-spend, read-only pre-settlement QA of the 6-run 2026-07-06 backed_depth divergence
cohort: verify complete, identity-safe, registry backed_depth, market-baselined,
cost-accounted, unsettled, and settlement-ready. No reconcile, no generation, no code change.

## 2. outcome

**COMPLETE -- QA PASSED, no blockers.** All 6 runs verified from the durable DB + read-only
endpoints. Registry backed_depth clean on all 6 (durable `PromptRouteProvenanceJson`); market
baselined (9 books each); 5 agree / 1 disagree confirmed (823036 divergence); 0 outcomes /
0 evaluations; all 6 SingleMatch reconcile-safe. Two minor NON-blocking notes: cost log is
session-scoped (non-durable; aggregate in capture report), market baseline is semi-structured
(agreement derived from LeanSide vs consensus, not a stored field).

## 3. repo state before / after

- dai before/after: `d79c38f` (unchanged), dirty only csproj phantom, 0/0.
- dai-vault before: `aa8c82f`, 2 ahead, untracked synopsis. after: `aa8c82f` + this QA docs
  commit, synopsis still untracked.

## 4. services used

DevCore.Api :5007 (read-only: reconcile-precheck, artifact, upcoming), devcore-sql (read-only
SQL). agent-service :8000 stayed DOWN (no generation, no model call). No paid services.

## 5. work performed

Phase 0 state (no drift) -> Phase 1 cohort membership from AgentRuns columns (6/6, no dups)
-> Phase 2 provenance from durable `PromptRouteProvenanceJson` (6/6 registry backed_depth,
no fallback, attribution complete, 64-char hash) -> Phase 3 market baseline from
`OutputJson.SourceDepth`/`SignalAvailability` (9 books each; 5 agree/1 disagree derived) ->
Phase 4 cost re-sum from captured stdout (6 gpt-4o-mini, $0.004259) -> Phase 5 settlement
readiness (0 outcomes/evals for cohort; 6/6 `/reconcile-precheck` SingleMatch) -> Phase 6
calibration projection -> Phases 7/8/9 docs + commit.

## 6. files changed

- dai: none.
- dai-vault (new, committed):
  - `06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-2026-07-06-v1.md`
  - `06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-handoff-2026-07-06-v1.md`

## 7. db writes / side effects

**None.** Read-only slice. AgentRuns 279, outcomes 112, evaluations 112 (all unchanged).
6 read-only `/reconcile-precheck` calls (write nothing) + read-only SQL + 1 read-only
`/artifact` fetch. No the-odds-api, no model calls.

## 8. paid calls / cost

0 paid calls, $0.00. (Re-verified the prior capture's 6 calls / $0.004259 from stdout; no new
spend.)

## 9. validation proof

6/6 runs present, 0 duplicates/superseded, all completed+active+NULL exclusion; 6/6 registry
+ fallback=false + backed_depth + attribution complete (durable column); market baselined
9 books each, agreement 5/1 matches known; 6 cost lines gpt-4o-mini total $0.004259; 0
outcomes + 0 evals for cohort; 6/6 SingleMatch reconcile-safe; no writes issued.

## 10. what did not change

runtime code, prompts, registry recipes, routing, confidence generation, calibration gate,
buyer copy, schema/migrations: unchanged. agent-service not started; `.env` untouched; dai
HEAD `d79c38f`. No settlement performed.

## 11. open issues

- Cohort unsettled -- awaiting 2026-07-06 official finals (all 6 pre-game at QA time, first
  pitch 14:10 ET).
- Cost evidence non-durable (session stdout); durable aggregate only in capture report v2.
- Market baseline semi-structured (agreement derived, not stored) -- fine for gamePk-based
  settlement; a structured market snapshot would help future automated edge computation.
- dai-vault unpushed: `aa8c82f` (capture v2) + `7fef9e8` (readiness) + this QA commit
  (PUSH_ALLOWED not granted).
- Services left running (devcore-sql, DevCore.Api :5007); agent-service down. Stop when done.

## 12. recommended next slice

**Backed-Depth Divergence Settlement / Reconciliation v1** -- settlement-only, after finals.
Settle 6 runs via identity `/reconcile` (all SingleMatch), write outcomes/evaluations,
re-read pooled calibration. Watch 823036.

## 13. suggested next prompt

```text
SLICE: Backed-Depth Divergence Settlement / Reconciliation v1
Date: <after 2026-07-06 finals available>  |  Mode: settlement-only, no paid generation.
SETTLEMENT_APPROVED=true, GENERATION_IN_THIS_SLICE=false, PUSH_ALLOWED=false.

Settle the QA-passed 6-run 2026-07-06 backed_depth divergence cohort (gamePks 822958, 822712,
824900, 823036, 823282, 823205) per backed-depth-divergence-cohort-integrity-qa-2026-07-06-v1.md.

Phase 0 verify: dai d79c38f (csproj phantom only); AgentRuns=279, outcomes=112, evaluations=112;
6 runs active (ExclusionReason NULL), SingleMatch (re-run /reconcile-precheck). Abort on drift.
Phase 1 services (read-only + reconcile): docker start devcore-sql; DevCore.Api :5007 health 200.
agent-service NOT needed (no model calls).
Phase 2 fetch official finals (StatsAPI) for all 6 gamePks; confirm all Final; record scores.
Phase 3 settle via identity POST /reconcile (all SingleMatch per QA); write outcomes + evaluations
ONLY for these 6. Record DAI correct/incorrect per run; special attention to divergence 823036
(DAI Cardinals home vs market Brewers away).
Phase 4 re-read pooled calibration (/metrics + /rows, ExclusionReason IS NULL filter); report
whether disagreement coverage/discrimination moved. Do NOT change the gate (expect still FALSE).
Hard constraints: no generation, no prompt/routing/confidence/buyer/gate/schema change; do not
push. End with a continuation-grade handoff.
```
