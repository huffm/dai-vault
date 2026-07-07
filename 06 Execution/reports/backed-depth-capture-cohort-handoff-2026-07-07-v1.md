---
title: "HANDOFF: Next Approved Backed-Depth Capture Cohort v1 -- CAPTURED 6/6 (2026-07-07)"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- 6-run cohort captured clean, 1 divergence; settlement pending after finals"
project: "DAI"
slice: "Next Approved Backed-Depth Capture Cohort v1"
related:
  - "06 Execution/reports/backed-depth-capture-cohort-2026-07-07-v1.md"
  - "06 Execution/reports/frozen-backed-depth-capture-slate-2026-07-07-v1.md"
  - "06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md"
---

# HANDOFF: next approved backed-depth capture cohort v1 (2026-07-07)

## 1. objective

capture one operator-approved MLB backed_depth cohort (<= 6 runs, <= $0.01 estimated)
under the cadence proposal: settleable close-favorite rows for gate-4
coverage/disagreement evidence. capture only; no reconciliation.

## 2. outcome

COMPLETE. 6/6 runs captured clean at 09:20-09:23 ET: all registry-routed
(promptSource=registry, zero fallbacks), selectedDataRegime == observedDataRegime ==
starter_enriched_market_backed_depth, attribution complete, 9 books each, identity
matched (mlb_statsapi + gamePk). **1 divergence captured: 824820 CHC@BAL (DAI home
Orioles 0.75 vs market away Cubs, gap 0.03) -- candidate edge signal only.** total
estimated cost $0.00423 (42% of the $0.01 ceiling). canary default-off restored and
verified. cohort is UNSETTLED by design; settlement after tonight's finals.

## 3. repo state before / after

- dai before and after: main @ `dbda7a8`, 0/0, dirty only on the phantom
  `DevCore.Data.csproj`. NO code changes, NO commit, NO push.
- dai-vault before: main @ `3e03a42`, 0/0, known untracked noise (manifest json,
  synopsis). after: 3 new slice docs committed + pushed (sha in session closeout); known
  noise still untracked, untouched.

## 4. services used

devcore-sql (up), DevCore.Api :5007 (up; source-readiness x6 + POST /api/agent-runs x6 +
/rows reads), agent-service :8000 (started AFTER slate freeze with
DAI_MLB_REGISTRY_PROMPT_CANARY=1 process-scoped inline env; stopped after capture; port
verified not listening; `.env` verified clean before and after). external: StatsAPI
schedule (free), the-odds-api (1 slate read + 6 readiness screens = 7 units; 386
remaining after the slate read).

## 5. work performed

- skills gate run (dai-skill-router; slice-runner doctrine + dai-agent-handoff +
  verification-before-completion).
- phase 0: baselines clean (dai `dbda7a8`, vault `3e03a42`, both 0/0); services verified;
  time 09:12 ET (parity with the validated 07-06 morning capture).
- phase 1-2: StatsAPI 16 games all Preview; odds slate read -> de-vig gaps -> EIGHT
  primary close favorites (0.03-0.07); six with both probables passed source-readiness
  6/6 (identity matched, predicted backed_depth, eligible); doubleheader excluded as
  Blocker; two starter-missing in-filter games held as unused backups.
- phase 3: slate FROZEN pre-spend (frozen-backed-depth-capture-slate-2026-07-07-v1.md;
  AgentRuns=279, outcomes/evals=118, 0 existing active runs per gamePk, canary verified
  off, guardrails re-verified from source: gpt-4o-mini @ sports_analyzer.py:647, metering
  present, allowlist includes backed_depth).
- phase 4-5: agent-service started canary process-scoped; run 1 (823687 CLE@MIN)
  generated alone and provenance-verified (registry, backed_depth, cost line) BEFORE
  runs 2-6; all 6 completed; run ids 9e2c433e/a32c433e/a92c433e/aa2c433e/ac2c433e/
  b32c433e (suffix -f36b-1410-8178-00373db4b724).
- phase 5-6: agent-service stopped; canary off verified; post-run verification: 1 active
  row per gamePk, all registry/backed_depth/attribution complete, outcome absent +
  evaluation 404 per run, rows 279 -> 285, settled 118 unchanged.
- phase 7-10: capture report, validation, vault commit, push.

## 6. files changed

dai: none.
dai-vault (one commit):
- `06 Execution/reports/frozen-backed-depth-capture-slate-2026-07-07-v1.md` (new)
- `06 Execution/reports/backed-depth-capture-cohort-2026-07-07-v1.md` (new)
- `06 Execution/reports/backed-depth-capture-cohort-handoff-2026-07-07-v1.md` (this file)

## 7. db writes / side effects

+6 AgentRuns (279 -> 285) via the established generation path only; 0 outcomes, 0
evaluations (118/118 unchanged); no direct SQL. side effects: agent-service started and
stopped; devcore-sql + DevCore.Api left running for the settlement slice. 7 the-odds-api
units consumed.

## 8. paid calls / cost

**6 paid gpt-4o-mini model calls; estimated total $0.00423** (per-run $0.000689-0.000726;
exactly 6 devcore.cost lines, one per run). under the 6-run and $0.01 approval caps.
metering is the public-list estimate from process stdout, not billing truth.

## 9. validation proof

- paid calls = 6 (cost-line count) and $0.00423 <= $0.01.
- 6/6 rows: promptSource=registry, legacyFallbackUsed=false, selected==observed==
  starter_enriched_market_backed_depth, attributionStatus=complete, sourceProvider=
  mlb_statsapi, externalGameId=gamePk, exclusionReason NULL, outcomeStatus null,
  evaluation 404.
- duplicate check: exactly 1 active row per selected gamePk (post-generation).
- registry: 6/6 routing log lines source=registry fallback=None; `.env` clean before and
  after; port 8000 not listening after stop.
- dai git status = phantom csproj only (no code change); no buyer copy touched.
- rows 285 = 279 + 6 exactly; settled 118 unchanged (no reconciliation).

## 10. what did not change

prompts, prompt registry recipes, routing logic, confidence generation, gates/thresholds/
criterion, models, buyer copy, schema/migrations, registry default-off posture, dai repo.
no settlement writes. no scheduler or background automation of any kind.

## 11. open issues

- cohort UNSETTLED: 6 runs await official finals (last first pitch 21:45 ET; finals
  expected ~01:00 ET 2026-07-08). settlement slice must re-verify finals fresh.
- if the divergence run settles, marketDisagreementN reaches 6 -- one short of the n=7
  re-projection checkpoint; the NEXT captured divergence triggers it.
- durable per-run cost sink still missing (cost evidence hand-recorded from stdout).
- services left running (devcore-sql, DevCore.Api); agent-service down as required.

## 12. recommended next slice

**Pre-Settlement QA / Settlement + Readout for this cohort** after all six games are
Final: strict preflight (gamePks 823687,824820,822956,822713,823280,824579; prefixes
9e2c433e,a32c433e,a92c433e,aa2c433e,ac2c433e,b32c433e), fresh StatsAPI finals, identity
/reconcile x6 with residue provenance, filled gate-4 evidence readout, and a check of the
disagreement checkpoint state. no-spend.

## 13. suggested next prompt

"SLICE: Backed-Depth Capture Cohort Settlement / Reconciliation (2026-07-07 cohort).
mode: settlement-only, no paid calls, no captures, writes only via POST /reconcile.
run after all 6 games Final (~01:00 ET 07-08; verify fresh). read first:
backed-depth-capture-cohort-2026-07-07-v1.md (section 10 has exact preflight args) +
gate4-evidence-readout-template-v1.md + the 07-07 settlement handoff pattern. phases:
repo/services baseline -> cohort identity confirm (6 runs, divergence 824820) -> fresh
statsapi finals gate (all 6 Final or stop) -> strict preflight exit 0 -> fresh /rows
before snapshot -> per-run re-read + identity /reconcile x6 (source=statsapi,
sourceRef=gamePk + final score, notes cohort+score+verification) -> post-settlement
verification -> filled gate-4 readout (gate4-evidence-readout-backed-depth-capture-
2026-07-07-v1.md) -> if marketDisagreementN >= 7, STOP capture cadence and run the
re-projection before any further capture -> docs commit + push after verification ->
13-section handoff."
