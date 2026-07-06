---
title: "HANDOFF: Backed-Depth Divergence Capture v2 (2026-07-06)"
type: "handoff"
date: "2026-07-06"
status: "COMPLETE -- 6-run backed_depth divergence cohort captured; settlement pending"
project: "DAI"
slice: "Backed-Depth Divergence Capture (PAID) v2"
related:
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-candidate-slate-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-capture-readiness-2026-07-05-v1.md"
---

# HANDOFF: Backed-Depth Divergence Capture v2 (2026-07-06)

## 1. objective

Morning execution of the paid backed-depth divergence capture: freeze a close-favorite
divergence slate, generate <= 12 registry-routed `backed_depth` MLB runs via the existing
app path, record every run + DAI-market agreement, respect all cost/model/route guardrails,
do NOT settle. Approval-gated (`PAID_CAPTURE_APPROVED=true`).

## 2. outcome

**COMPLETE.** 6-run cohort captured (all 6 selected close favorites generated). Registry
route clean on all 6 (no fallback), model `gpt-4o-mini`, one call each, total est. cost
**$0.004259** (< $0.05). DAI-market: **5 agree, 1 disagree** -- the first readable
backed_depth divergence (823036 MIL@STL: DAI home Cardinals vs market away Brewers, gap
0.042). No settlement, no code/prompt/routing/gate change, nothing pushed.

## 3. repo state before / after

- dai before: `d79c38f`, dirty only csproj phantom, 0/0. after: `d79c38f` (unchanged), same
  phantom, 0/0.
- dai-vault before: `7fef9e8`, 1 ahead (readiness commit), untracked synopsis. after:
  `7fef9e8` + this slice's docs commit (3 new report files), synopsis still untracked.

## 4. services used

Docker Desktop (started from down); devcore-sql (started from Exited); DevCore.Api :5007
(built+run, Development); agent-service :8000 (started after freeze with canary
process-scoped, stopped after capture). sports-app not started.

## 5. work performed

Phase 0 state capture (no drift) -> Phase 1 window confirm (08:37 ET, all 8 pre-game, 5.5h
margin; >=4 candidates -> generation permitted) -> guardrails verified from source
(gpt-4o-mini, single create(), cost log, canary default-off, allowlist includes
backed_depth) -> Phase 4 slate build (StatsAPI 8 games; 1 the-odds-api h2h read; 6
source-readiness screens) -> Phase 5 slate FROZEN (AgentRuns=273) -> Phase 6 agent-service
started with `DAI_MLB_REGISTRY_PROMPT_CANARY=1` process-scoped -> Phase 7 generated 6 runs
(run 1 verified registry route before continuing) -> Phase 8 agent-service stopped,
default-off restored -> Phases 9/12/13 docs + commit.

## 6. files changed

- dai: none (csproj phantom pre-existing).
- dai-vault (new, committed):
  - `06 Execution/reports/backed-depth-divergence-candidate-slate-2026-07-06-v2.md`
  - `06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md`
  - `06 Execution/reports/backed-depth-divergence-capture-handoff-2026-07-06-v2.md`

## 7. db writes / side effects

- +6 AgentRuns (273 -> 279), all `completed`, `ExclusionReason=NULL`, ExternalGameId =
  StatsAPI gamePk. 0 outcomes, 0 evaluations (112 -> 112 both).
- gamePks/runIds: 822958 ac31433e; 822712 ad31433e; 824900 b331433e; 823036 b431433e;
  823282 b631433e; 823205 b731433e (all suffix `-f36b-1410-8175-00373db4b724`).
- External: 6 paid gpt-4o-mini calls; ~7 the-odds-api units (screening).

## 8. paid calls / cost

6 paid model calls; estimated total **$0.004259** (per-run $0.00070-0.00074). Under both the
12-run and $0.05 caps.

## 9. validation proof

AgentRuns +6 exactly; outcomes/evaluations unchanged at 112; 6 cost lines all gpt-4o-mini
one-per-run; all 6 `promptSource=registry` + `legacyFallbackUsed=false` +
regime `starter_enriched_market_backed_depth`; `.env` has no canary; `git status` dai = only
csproj phantom; HEAD `d79c38f` unchanged; nothing pushed.

## 10. what did not change

runtime code, prompts, prompt registry recipes, routing, confidence generation, buyer copy,
calibration gate, reconciliation logic, schema/migrations: unchanged. registry default-off
restored (canary process-scoped only; `.env` untouched).

## 11. open issues

- 6 runs are captured but **unsettled** -- outcomes/evaluations pending official finals.
- Gate 4 still FALSE on merits (`discrimination_inverted`,
  `insufficient_market_disagreement`, `insufficient_market_coverage`); this cohort grows the
  disagreement evidence (1 new divergence) but does not by itself flip the gate.
- dai-vault docs commit is unpushed (PUSH_ALLOWED=false); readiness commit `7fef9e8` also
  still unpushed.
- Services left running: devcore-sql + DevCore.Api :5007 (agent-service stopped, Docker up).
  Stop when convenient; not required for correctness.

## 12. recommended next slice

**Backed-Depth Divergence Settlement / Reconciliation v1** -- after 2026-07-06 finals,
settle the 6 runs (identity `/reconcile`), write outcomes/evaluations, then re-read pooled
calibration. Settlement-only; no generation, no gate tuning.

## 13. suggested next prompt

```text
SLICE: Backed-Depth Divergence Settlement / Reconciliation v1
Date: <after 2026-07-06 finals available>  |  Mode: settlement-only, no paid generation.
SETTLEMENT_APPROVED=true, GENERATION_IN_THIS_SLICE=false, PUSH_ALLOWED=false.

Settle the 6-run 2026-07-06 backed_depth divergence cohort (gamePks 822958, 822712, 824900,
823036, 823282, 823205) captured in backed-depth-divergence-capture-2026-07-06-v2.md.

Phase 0 verify state: dai d79c38f (csproj phantom only); AgentRuns=279, outcomes=112,
evaluations=112; confirm the 6 runs active (ExclusionReason NULL). Abort on drift.
Phase 1 services (read-only + reconcile): docker start devcore-sql; DevCore.Api :5007 health
200. agent-service NOT needed (no model calls).
Phase 2 pre-check: for each gamePk run GET /api/agent-runs/reconcile-precheck to choose
identity /reconcile (SingleMatch) vs per-run /{id}/outcome (MultipleMatches).
Phase 3 fetch official finals (StatsAPI) for all 6 gamePks; confirm all Final.
Phase 4 settle via the established /reconcile residue contract; write outcomes + evaluations
ONLY for these 6. Record DAI correct/incorrect per run, with special attention to the
divergence run 823036 (DAI Cardinals vs market Brewers).
Phase 5 re-read pooled calibration (/metrics + /rows, ExclusionReason IS NULL filter); report
whether disagreement coverage/discrimination moved. Do NOT change the gate.
Hard constraints: no generation, no prompt/routing/confidence/buyer/gate/schema change; do
not push. End with a continuation-grade handoff.
```
