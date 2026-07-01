---
title: "Live Batch + Settlement Reconciliation Gate v1"
type: "reconciliation"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Live Batch + Settlement Reconciliation Gate v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - calibration
  - capture
related:
  - "06 Execution/reconciliations/outcome-reconciliation-readiness-next-slice-2026-06-30.md"
---

# Live Batch + Settlement Reconciliation Gate v1 -- Evidence Report

**status:** complete (prior soak runs PENDING reconciliation; 8-game live batch run + verified)
**date:** 2026-06-29

## purpose

Two operational steps: (1) gate outcome reconciliation on the 3 prior live-soak runs (only reconcile settled
games), and (2) run a controlled live MLB batch through the real .NET -> FastAPI path with prompt route
provenance persisted and visible, complementing the starter-missing soak by exercising the enriched allowlisted
regimes live.

## prior live-soak push status

The prior live-soak doc commit (dai-vault `ba22e71`,
`docs(execution): record live-scheduled starter-missing soak evidence`) was already pushed in the previous turn;
confirmed on origin/main, dai-vault clean and synced. No push needed at this slice's start.

## service status

`devcore-sql` already up on 1433. Brought up FastAPI agent-service (uvicorn :8000, canary on) and .NET platform
API (`dotnet run` :5007); health ok (agent `/api/ping`, api listening). Both stopped after the batch (canary env
dropped); devcore-sql left up.

## db / migration status

`AgentRuns.PromptRouteProvenanceJson` present + nullable (migration `20260629174632` applied a prior slice). New
batch runs wrote non-null provenance.

## reconciliation gate (3 prior live-soak runs)

| run | game | gamePk | game status | AgentRunOutcome | eligible? |
|---|---|---|---|---|---|
| RUN1 1fbd433e | Marlins @ Rockies (06-30) | 824338 | Scheduled / Preview | 0 | NO (future) |
| RUN2 25bd433e | Giants @ Dbacks (06-30) | 825066 | Scheduled / Preview | 0 | NO (future) |
| RUN3 28bd433e | White Sox @ Orioles (07-01) | 824818 | Scheduled / Preview | 0 | NO (future) |

All three games are still `Scheduled` / `Preview` (not final) as of 2026-06-29; all three runs have 0
`AgentRunOutcome` rows and `ExclusionReason = null` (active). **PENDING -- not reconciled.** No outcomes were
fabricated and no future game was reconciled. Match-rate for these routes will be populated by a later
outcome-reconciliation pass once the games settle.

## live schedule probe

`statsapi /api/v1/schedule?date=2026-06-29&hydrate=probablePitcher`: 13 games, all both-announced (0 TBD-starter
today). So today's batch exercises the enriched allowlisted regimes (the starter-missing regimes are not
naturally available today). 8 games selected for the batch (within the 5-8 cap).

## candidate games / live batch plan + approval

Operator approved an **8-run** batch ("Approve 8 runs"). Games (all both-announced -> starter_enriched
candidates; market sub-regime per live odds at run time):

White Sox@Orioles, Pirates@Phillies, Tigers@Yankees, Mets@Blue Jays, Nationals@Red Sox, Rangers@Guardians,
Reds@Brewers, Padres@Cubs (all 2026-06-29).

## live batch execution

8 runs via `POST /api/agent-runs` (full chain). **8 paid gpt-4o-mini calls** (one per run; no retries -- agent
log shows exactly 8 "mlb analysis response" + 8 "mlb prompt routing decision" lines). Approx cost well under
$0.02. All 8 returned status=completed.

## run evidence

All 8 derived regime `starter_enriched_market_backed_depth` (announced starters + multi-book odds). **7 of 8
routed registry-authoritative; 1 hit the safe fail-closed fallback:**

| run | agentRunId | game | promptSource | regime | recipe@ver | regAuth | legacyFB | fallbackReason |
|---|---|---|---|---|---|---|---|---|
| RUN1 | 2dbd433e | White Sox@Orioles | registry | starter_enriched_market_backed_depth | ...backed_depth.v1@v1 | true | false | null |
| RUN2 | 33bd433e | Pirates@Phillies | registry | starter_enriched_market_backed_depth | ...backed_depth.v1@v1 | true | false | null |
| RUN3 | 36bd433e | Tigers@Yankees | **live** | starter_enriched_market_backed_depth | (none) | true | **true** | **assembly_error** |
| RUN4 | 37bd433e | Mets@Blue Jays | registry | starter_enriched_market_backed_depth | ...backed_depth.v1@v1 | true | false | null |
| RUN5 | 3dbd433e | Nationals@Red Sox | registry | starter_enriched_market_backed_depth | ...backed_depth.v1@v1 | true | false | null |
| RUN6 | 42bd433e | Rangers@Guardians | registry | starter_enriched_market_backed_depth | ...backed_depth.v1@v1 | true | false | null |
| RUN7 | 48bd433e | Reds@Brewers | registry | starter_enriched_market_backed_depth | ...backed_depth.v1@v1 | true | false | null |
| RUN8 | 4cbd433e | Padres@Cubs | registry | starter_enriched_market_backed_depth | ...backed_depth.v1@v1 | true | false | null |

RUN3 is the byte-equality / assembly fail-closed path working as designed: the regime derived to an allowlisted
one, but the registry recipe could not assemble byte-identically for that game's partial evidence shape, so the
canary fell back to the LIVE prompt (`promptSource=live`, `legacyFallbackUsed=true`, `fallbackReason=assembly_error`).
The run still completed normally on the live prompt; no error surfaced to the buyer path. This is the safety
envelope behaving correctly -- the model never received a non-live-equivalent prompt.

## durable persistence evidence

dev-SQL: of the 8 batch rows, `prov_not_null = 8` (all persisted, including the live-fallback run);
`src=registry` x7, `src=live` x1. `GET /api/agent-runs/{id}/artifact` returned the same `promptRouteProvenance`
for each (the table above was read from that endpoint).

## metrics endpoint evidence

`GET /api/agent-runs/prompt-route-calibration/metrics`: `totalRows=246` (238 + 8), `registryRows=10`
(3 prior starter-missing soak + 7 new enriched), `liveRows=1` (the assembly_error fallback run). Route
`...starter_enriched_market_backed_depth.v1@v1::starter_enriched_market_backed_depth` -> total=7, registry=7. The
live-fallback run has a null recipe/version, so its `promptRouteKey` is "unknown" (correct) and it is not in the
enriched route row.

## tests run

Pre-paid: `dotnet build DevCore.Api.Tests` -> 0 errors; `dotnet test ... --filter
"PromptRouteProvenance|PromptRouteCalibration|AgentRunsController"` -> 90 passed; `python
scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes). Post-paid: DB + `/artifact` + metrics endpoint
verification (above). No code changed, so no suite re-run.

## paid calls

8 (one gpt-4o-mini call per run; no retries). Approx well under $0.02 total.

## buyer-facing impact

None. `SportsAnalysisResponse` body unchanged; no buyer copy/UX change; no chain-of-thought / prompt text /
prompt bytes exposed.

## default allowlist status

Unchanged at four regimes.

## risks / deferred items

- The 3 prior starter-missing soak runs remain PENDING reconciliation (future games); an outcome-reconciliation
  pass is needed once they settle (06-30 / 07-01).
- This batch also produced future/in-progress-game runs (today's slate); their outcomes are likewise pending.
- All 8 batch games derived to `starter_enriched_market_backed_depth` (today's slate had announced starters +
  multi-book odds); `starter_enriched_market_missing` was not naturally exercised this batch.
- RUN3's `assembly_error` fallback is worth a follow-up look (which partial-evidence field blocked byte-identical
  assembly for that game) -- not a defect (safe fallback), but a candidate for an overlay-coverage diagnostic.
- Staging/prod still need the provenance migration applied on deploy. Canary stays DEFAULT OFF normally (enabled
  via env only for this batch, dropped when services stopped).

## rollback / cleanup posture

No rollback needed. The 8 runs are legitimate additive AgentRun rows in dev (no backfill, no reset). Canary env
was process-local and dropped when the services were stopped; devcore-sql left up. No code/schema change.

## next recommended slice

Outcome Reconciliation Pass for Live Batch v1 (reconcile the prior soak + this batch once games settle, populate
match-rate), OR Broad Cohort Rerun Grouped by Prompt Recipe v1, OR Calibration Metrics Export Download v1.

Related: [[live-scheduled-starter-missing-soak-v1]], [[thin-tenant-scoped-calibration-metrics-endpoint-v1]],
[[calibration-outcome-metrics-by-prompt-route-v1]], [[apply-agentrun-provenance-migration-to-dev-sql-v1]].
