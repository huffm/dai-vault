---
title: "Live-Scheduled Starter-Missing Soak v1"
type: "evidence-report"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Live-Scheduled Starter-Missing Soak v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - source-depth
  - capture
related:
  - "06 Execution/reports/starter-missing-registry-canary-confirmation-v1.md"
---

# Live-Scheduled Starter-Missing Soak v1 -- Evidence Report

**status:** complete (both target regimes confirmed registry-authoritative on genuine live-scheduled games)
**date:** 2026-06-29

## purpose

Confirm the two promoted starter-missing prompt regimes through the REAL live-scheduled .NET -> FastAPI analyzer
path on genuine scheduled MLB games (not fixtures), with prompt route provenance persisted to the durable
AgentRun row and visible through the read model + metrics endpoint. This closes the gap left by the earlier
fixture-based canary confirmation.

## start state

dai `289777f` (main, synced, on origin); dai-vault `b0fc720` (main, synced, on origin). `devcore-sql` up on
1433 with migration `20260629174632_AddAgentRunPromptRouteProvenance` applied (verified prior slice).
`DEFAULT_ALLOWLIST` = four regimes (both starter-missing targets included). Services :5007/:8000 were down at
start. Only the pre-existing untracked `06 Execution/system-state-synopsis-v1.md` dirty (untouched).

## service status

Brought up via the established dev workflow: FastAPI agent-service `python -m uvicorn main:app --port 8000`
(canary on) and .NET platform API `dotnet run` (ASPNETCORE_ENVIRONMENT=Development, :5007); `devcore-sql`
already up. Health: agent `GET /api/ping` -> ok; platform API listening :5007. Services stopped after the soak
(canary env dropped with the processes); devcore-sql left up.

## db / migration status

`AgentRuns.PromptRouteProvenanceJson` present + nullable in dev SQL (migration applied prior slice). New runs
wrote non-null provenance to it (evidence below).

## target regimes

- `starter_missing_market_missing`
- `starter_missing_market_backed_depth`

## candidate game selection (free statsapi probe, no model call)

`statsapi /api/v1/schedule?sportId=1&hydrate=probablePitcher` for upcoming dates. Genuine live-scheduled games
with a TBD probable starter (-> no MlbStarterContext -> starter_missing):

| gamePk | date | game | starter | selected |
|---|---|---|---|---|
| 824338 | 2026-06-30 | Miami Marlins @ Colorado Rockies | home TBD | yes (RUN1) |
| 825066 | 2026-06-30 | San Francisco Giants @ Arizona Diamondbacks | home TBD | yes (RUN2) |
| 824818 | 2026-07-01 | Chicago White Sox @ Baltimore Orioles | both TBD | yes (RUN3) |

2026-06-29 had 0 TBD-starter games (all 13 both-announced), so the candidates are the 06-30/07-01 TBD games. The
market sub-regime was determined by the platform's odds retrieval at run time (not forced).

## config / env used

`DAI_MLB_REGISTRY_PROMPT_CANARY=1` on the agent-service (registry-authoritative path enabled; DEFAULT_ALLOWLIST
covers both targets -- no per-regime override used). `DAI_MLB_ROUTE_PROVENANCE_PATH` set to a scratch audit sink.
`OPENAI_API_KEY` from `services/agent-service/.env`. .NET dev auth bypass on (no bearer token).

## paid-call approval + execution

Operator approved "up to 3 runs", "you run it". Executed **3** live-scheduled runs via `POST /api/agent-runs`
(full chain: retrieve -> one analyze call -> persist). **3 paid gpt-4o-mini calls total** (one per run; no
retries / no second call -- agent log shows exactly 3 "mlb analysis response" + 3 "mlb prompt routing decision"
lines). Approx cost: well under $0.01 total. OpenAI quota was available (all 3 completed).

## run evidence

All three completed (status=completed) and routed registry-authoritative; both target regimes covered live
(backed_depth via RUN1, missing via RUN2/RUN3):

| run | agentRunId | game | selectedDataRegime | promptSource | regAuth | legacyFallback | allowlisted | fallback | recipe@ver | assembledHash (prefix) |
|---|---|---|---|---|---|---|---|---|---|---|
| RUN1 | 1fbd433e-f36b-1410-816e-00373db4b724 | Marlins @ Rockies (06-30) | starter_missing_market_backed_depth | registry | true | false | true | null | mlb.pregame.analysis.starter_missing_market_backed_depth.v1@v1 | 5d24dd60 |
| RUN2 | 25bd433e-f36b-1410-816e-00373db4b724 | Giants @ Dbacks (06-30) | starter_missing_market_missing | registry | true | false | true | null | mlb.pregame.analysis.starter_missing_market_missing.v1@v1 | 1a481863 |
| RUN3 | 28bd433e-f36b-1410-816e-00373db4b724 | White Sox @ Orioles (07-01) | starter_missing_market_missing | registry | true | false | true | null | mlb.pregame.analysis.starter_missing_market_missing.v1@v1 | b4e6a168 |

Every objective-11 field is satisfied for all three: promptSource=registry, registryAuthoritativeEnabled=true,
legacyFallbackUsed=false, fallbackReason=null, regimeAllowlisted=true, selectedDataRegime=target,
selectedPromptRecipeId + selectedPromptVersion + assembledHash populated.

## durable persistence evidence

Direct dev-SQL query (`devcore` DB) for the three agentRunIds:
```
1FBD433E-...|prov_not_null=1|regime=starter_missing_market_backed_depth|src=registry
25BD433E-...|prov_not_null=1|regime=starter_missing_market_missing|src=registry
28BD433E-...|prov_not_null=1|regime=starter_missing_market_missing|src=registry
```
`GET /api/agent-runs/{id}/artifact` returned the same `promptRouteProvenance` for each (the table above was read
from that endpoint). Full path proven end to end: FastAPI X-Prompt-Route-Provenance header -> .NET capture ->
`AgentRuns.PromptRouteProvenanceJson` -> `/artifact` read model.

## metrics endpoint evidence

`GET /api/agent-runs/prompt-route-calibration/metrics` (tenant-scoped): summary `totalRows=238`
(235 historical + 3 new), `registryRows=3` (the 3 new live runs). Both target routes present:
- `mlb.pregame.analysis.starter_missing_market_missing.v1@v1::starter_missing_market_missing` -> totalRows=2, registryRows=2
- `mlb.pregame.analysis.starter_missing_market_backed_depth.v1@v1::starter_missing_market_backed_depth` -> totalRows=1, registryRows=1

The new runs are unreconciled (future games, no outcome yet), so they sit in unreconciledRows and do not affect
matchRate -- correct per the metrics contract.

## blocked / missing target regimes

None. Both target regimes were produced naturally by genuine live-scheduled games (no fabrication, no
fixture-only data). RUN1's game had multi-book odds depth at run time -> backed_depth; RUN2/RUN3 had no posted
market -> missing.

## buyer-facing impact

None. `SportsAnalysisResponse` body shape unchanged; no buyer copy/UX change. Provenance lives on the run-row
column + internal read model only; no chain-of-thought / prompt text / prompt bytes exposed.

## default allowlist status

Unchanged at four regimes.

## tests run

Pre-paid: `dotnet build DevCore.Api.Tests` -> 0 errors; `dotnet test ... --filter
"PromptRouteProvenance|PromptRouteCalibration|AgentRunsController"` -> 90 passed; `python
scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes). Post-paid: DB + `/artifact` + metrics endpoint
verification (above). No code changed, so no suite re-run needed.

## risks / deferred items

- The three runs are unreconciled (future games); outcome reconciliation + match-rate for these routes comes
  later once results land. A later slice could reconcile and re-read the metrics.
- Live confirmation used the dev database/stack; staging/prod still need the migration applied on deploy.
- Market sub-regime depends on live odds at run time; today's draw happened to cover both -- not guaranteed on
  every future slate.
- Canary remains DEFAULT OFF in normal operation; it was enabled only for this soak via env and dropped when the
  services were stopped (no persistent config change).

## rollback / cleanup posture

No rollback needed. The 3 runs are legitimate additive AgentRun rows in dev (no backfill, no data reset). Canary
env was process-local and dropped when the agent-service + platform API were stopped after the soak;
`devcore-sql` left up. No code or schema change.

## next recommended slice

Broad Cohort Rerun Grouped by Prompt Recipe v1, OR Calibration Metrics Export Download v1, OR an outcome-
reconciliation pass to populate match-rate for the newly captured starter-missing routes once these games settle.

Related: [[apply-agentrun-provenance-migration-to-dev-sql-v1]],
[[dotnet-agentrun-prompt-provenance-persistence-v1]], [[thin-tenant-scoped-calibration-metrics-endpoint-v1]],
[[starter-missing-registry-canary-confirmation-v1]], [[default-allowlist-widening-v1]].
