# Fresh Artifact Generation Operability Fix v1

**date:** 2026-06-09
**status:** operability diagnosis. dev-docs only change in `dai`. no prompt change, no parser change, no decision/confidence/posture/lean change, no schema change, no source addition, no probe-refresh activation, no .NET/FastAPI/Angular runtime code change.
**scope:** make the local fresh sports artifact generation path operable, diagnosable, and documented. fix the factory line controls, not the product unit.

## Purpose

DAI is a decision artifact factory. Buyer Artifact Quality Check Pass v1 could not generate fresh artifacts: the platform API timed out and the agent service refused connection. This slice determines whether that blocker is a buyer-artifact quality problem (it is not) or a local runtime operability problem (it is), and isolates the exact remaining blockers.

Primary question: can a developer start the required local services and generate at least one fresh sports decision artifact through the intended local path?

## Scope

- read handoff and relevant vault/dev docs
- map the local generation service topology
- review startup scripts, ports, base URLs, env, and config for drift
- start the local services with the existing documented commands
- attempt the smallest reasonable fresh artifact generation
- isolate the exact failing layer if generation still fails
- document startup, smoke, and the remaining blockers

## Non-goals

- no FastAPI prompt edits
- no parser behavior change
- no decision/confidence/posture/lean/artifact-semantics change
- no source addition, no probe-refresh activation
- no schema change
- no auth/billing/tenants/dashboards/Kubernetes/deployment work
- no Jera work
- no push

## Observed blocker (from Buyer Artifact Quality Check Pass v1)

- platform API on `localhost:5007` did not respond within the timeout
- FastAPI agent service on `127.0.0.1:8000` refused connection

Interpretation going in: probably local operability, not artifact quality. Confirmed.

## Service topology found

The intended local sports generation path is three services plus a database:

| component | what it is | start command | host/port | health |
|---|---|---|---|---|
| agent service | FastAPI analyzer (the "agent service" that refused connection) | `scripts/start-agent-service.ps1` (uvicorn) | `http://127.0.0.1:8000` | `GET /api/ping` -> `{"status":"ok"}` |
| platform API | .NET `DevCore.Api` (the "platform API" that timed out) | `scripts/start-platform-api.ps1` (`dotnet run`) | `http://localhost:5007` | `GET /health` -> `{"status":"ok"}` |
| sports app | Angular thin frontend | started by `scripts/dev/sports/start-sports-dev.ps1` | `http://127.0.0.1:4201` | n/a |
| SQL | `devcore` database in the `devcore-sql` Docker container | Docker Desktop + container | `localhost,1433` | TCP connect |

Call direction:

- Angular sports app -> .NET platform API (`POST /api/agent-runs`)
- .NET platform API -> FastAPI analyzer (`AiService:BaseUrl` -> `POST /api/sports/analyze`)
- .NET platform API -> SQL (`ConnectionStrings:Sql`) for identity resolution and `AgentRun` persistence

Where the full buyer artifact comes from: the persisted v3 artifact (with `cognitiveProtocol`) is assembled and persisted by the .NET layer, so a fresh buyer-facing artifact requires the full chain (Angular/script -> .NET -> FastAPI -> SQL). The direct FastAPI `POST /api/sports/analyze` is a lower-level analyzer read only (needs the model key, not SQL).

One-shot launcher for the sports stack: `scripts/dev/sports/start-sports-dev.ps1`. Smoke harness: `scripts/dev/sports/test-sports-dev.ps1` (5 checks). Fresh-artifact batch + vault report: `scripts/dev/sports/run-artifact-calibration.ps1`.

## Configuration findings

No drift was found. Every cross-service reference is internally consistent:

- platform API port `5007` is consistent across `DevCore.Api/Properties/launchSettings.json` (http profile `applicationUrl`), `scripts/dev/sports/README.md`, and `test-sports-dev.ps1` (`$netApiBase`).
- `start-platform-api.ps1` runs `dotnet run` with no profile, so it binds the first profile (`http`) = `localhost:5007`. Correct.
- `AiService:BaseUrl` in `appsettings.Development.json` = `http://127.0.0.1:8000`, exactly the uvicorn bind. No HTTP/HTTPS or trailing-path mismatch.
- `Dev:EnableBypassAuth: true` in `appsettings.Development.json`, so the local dev API needs no bearer token.
- `services/agent-service/.env` contains `OPENAI_API_KEY` (present, not missing).
- the agent-service `.venv` is healthy and correctly located: `pyvenv.cfg` was created in place at the current `dai-workspace\dai\services\agent-service\.venv` path, and `Scripts/` contains `python.exe` and `uvicorn.exe`. The documented "stale venv from repo move" failure mode does not apply here.
- `import main` from the venv succeeds (`fastapi, uvicorn, openai, dotenv` all import; app = "DevCore AI Backend" 0.3.0).

## Startup commands tested

| service | command used | started | port bound | health | result |
|---|---|---|---|---|---|
| agent service | `.\.venv\Scripts\python.exe -m uvicorn main:app --host 127.0.0.1 --port 8000` (the exact command in `start-agent-service.ps1`) | yes | 127.0.0.1:8000 | `GET /api/ping` -> `{"status":"ok"}` | healthy; all routes mounted incl. `/api/sports/analyze` |
| platform API | `ASPNETCORE_ENVIRONMENT=Development dotnet run --project .\DevCore.Api\DevCore.Api.csproj` (the exact command in `start-platform-api.ps1`) | yes | localhost:5007 | `GET /health` -> `{"status":"ok"}` | healthy; "Now listening on: http://localhost:5007" |

Both services that "refused connection" / "timed out" during the quality pass start cleanly with the existing documented commands and pass their health checks. Before this slice, nothing was listening on 8000, 5007, 4201, or 1433 -- consistent with the services simply not having been started.

## Smoke generation result

Two generation paths were exercised.

1. Direct analyzer path (lower-level, no SQL):
   - request: `POST http://127.0.0.1:8000/api/sports/analyze` with an NBA matchup (`Boston Celtics` vs `Denver Nuggets`, `2026-06-12`), header `X-Agent-Run-Id: smoke-nba-0609`.
   - result: HTTP 500. uvicorn log shows the request reached the model call and OpenAI returned `429 insufficient_quota`: "You exceeded your current quota, please check your plan and billing details." (`openai.RateLimitError`, `code: insufficient_quota`).
   - meaning: the analyzer worker runs and dispatches correctly; the model provider refused the call because the API key has no remaining quota/billing.

2. Full platform path (intended buyer-artifact path, needs SQL):
   - request: `POST http://localhost:5007/api/agent-runs` with `runType: sports.matchup.analysis` and the same NBA input.
   - result: HTTP 500 from a `Microsoft.Data.SqlClient.SqlException` -- "A network-related or instance-specific error occurred ... The wait operation timed out." The stack fails at `IdentityResolver.ResolveDevelopmentAsync` (`IdentityResolver.cs:138`), i.e. at the first SQL touch (identity resolution), before the analyzer/model is even reached.
   - meaning: this SQL connection timeout is exactly the "platform API timed out" symptom. It is caused by the `devcore-sql` container not running because the Docker daemon is not running.

No fresh artifact was produced. No artifact JSON was saved.

## Ports / endpoints verified

- `127.0.0.1:8000` `GET /api/ping` -> 200 `{"status":"ok"}` (agent service)
- `127.0.0.1:8000` `POST /api/sports/analyze` -> 500 (model quota, see above)
- `localhost:5007` `GET /health` -> 200 `{"status":"ok"}` (platform API)
- `localhost:5007` `POST /api/agent-runs` -> 500 (SQL not reachable, see above)

## Is fresh artifact generation now operable?

Line controls: yes. Both services start, bind the correct ports, and pass health checks using the existing documented commands. There is no port drift, no base-URL drift, no missing/mismatched env var, no stale venv, and no startup/config code bug.

End-to-end fresh artifact: no, not yet. Two environment/account dependencies are currently down, and both are outside this slice's code scope:

1. OpenAI quota exhausted -- `429 insufficient_quota` on the key in `services/agent-service/.env`. This blocks the direct analyzer path and would block the full chain at the model step. This is the binding blocker for producing any fresh read.
2. Docker daemon / `devcore-sql` not running -- the full `/api/agent-runs` chain fails at identity resolution with a SQL connection timeout (the "platform API timed out" symptom).

Neither is a DAI code or config defect. The factory line controls are turnable; the line is currently starved of two external inputs (a funded model key and the SQL container).

## Remaining blocker and next fix (exact, narrow)

To actually generate a fresh sports artifact, in order:

1. Restore a funded model key: set a working `OPENAI_API_KEY` (top up billing on the current key or swap to a funded key) in `services/agent-service/.env`. Account/billing action; no code change.
2. Start SQL: start Docker Desktop, then start the `devcore-sql` container so `localhost,1433` accepts connections (the dev connection string and the `docker exec -i devcore-sql ...` reference in `scripts/dev/sports/purge-dev-agent-runs.ps1` both target this container).
3. Re-run the documented path below and confirm.

## Reproduction (once the model key is funded and SQL is up)

From the `dai` repo root:

```powershell
# 1. start the sports stack (agent service 8000, platform api 5007, sports app 4201)
powershell -ExecutionPolicy Bypass -File .\scripts\dev\sports\start-sports-dev.ps1
# wait ~15s for all three windows to finish starting

# 2. smoke the stack (5 checks; check 5 is the full chain and needs SQL)
powershell -ExecutionPolicy Bypass -File .\scripts\dev\sports\test-sports-dev.ps1

# 3. generate a small fresh batch + vault report (dry run first)
pwsh -File .\scripts\dev\sports\run-artifact-calibration.ps1 -Competition nba -Take 3 -DryRun
pwsh -File .\scripts\dev\sports\run-artifact-calibration.ps1 -Competition nba -Take 3
```

Lower-level isolation checks used in this slice (no full stack required):

```powershell
# agent service only
cd services\agent-service; .\.venv\Scripts\python.exe -m uvicorn main:app --host 127.0.0.1 --port 8000
# then, from another shell:
#   GET  http://127.0.0.1:8000/api/ping              -> {"status":"ok"}
#   POST http://127.0.0.1:8000/api/sports/analyze    -> 500 today (429 insufficient_quota); 200 once the key is funded
```

## Files changed

- `dai-vault/04 Products/sports-v1/fresh-artifact-generation-operability-fix-v1.md` (new -- this report)
- `dai-vault/06 Execution/handoffs/current-slice.md` (addendum for this slice)
- `dai/scripts/dev/sports/README.md` (troubleshooting: model-quota `429 insufficient_quota` failure mode; Docker Desktop + `devcore-sql` must be running for the full chain)

No `.NET`, FastAPI, Angular, prompt, parser, schema, or decision-logic file was changed.

## Verification performed

- repo baseline recorded for `dai`, `dai-vault`, `jera-workspace-skills` (jera not present in this workspace)
- listening-port scan before startup: nothing on 8000/5007/4201/1433
- venv health: `pyvenv.cfg` in-place, `Scripts/python.exe` and `uvicorn.exe` present, `import main` succeeds
- agent service started; `GET /api/ping` -> 200 `{"status":"ok"}`
- direct analyzer `POST /api/sports/analyze` -> 500, isolated to `openai.RateLimitError 429 insufficient_quota`
- platform API started; `GET /health` -> 200 `{"status":"ok"}`
- full chain `POST /api/agent-runs` -> 500, isolated to SQL connection timeout at `IdentityResolver.ResolveDevelopmentAsync`
- docs-only verification for the README/vault changes: `git diff --check`, added-line exact-path scan, vault non-ASCII added-line scan

## Deferred decisions / ledger

No ledger update was needed. No new deferred runtime, prompt, source, or artifact-contract decision was discovered. The two remaining blockers are an exhausted model-key quota (billing/account) and a stopped SQL container (local Docker), neither of which is a consciously deferred runtime design decision.

## What was not changed

- no FastAPI prompt, route, or model code
- no parser / sanitation behavior
- no .NET runtime logic, controller, or composer
- no confidence/posture/lean/decision/artifact-semantics change
- no schema or migration
- no source expansion, no probe-refresh activation
- no Angular code
- no auth/billing/tenant/dashboard/Kubernetes/deployment work
- no Jera work
- no `OPENAI_API_KEY` value change and no secret written to the repo
