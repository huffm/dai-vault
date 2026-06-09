# Post-Credit Fresh Artifact Generation Smoke v1

**date:** 2026-06-09
**status:** operability smoke + read-only inventory. dev-docs only change in `dai` (one troubleshooting line). no prompt/buyer-copy/confidence/posture/lean/signal/decision change, no source addition, no probe-refresh activation, no schema change, no cost-guardrail or pricing work, no .NET/FastAPI/Angular runtime code change.
**scope:** turn the local factory line back on now that OpenAI credits were added, prove whether a fresh sports artifact can be produced end-to-end, and inventory where model spend enters the artifact factory.

## Purpose

Fresh Artifact Generation Operability Fix v1 found the line controls healthy but fresh generation blocked by two environmental conditions: an exhausted OpenAI key quota (`429 insufficient_quota`) and a stopped `devcore-sql` container (SQL timeout in the full chain). The user added OpenAI credits. This slice reruns the smoke, brings up SQL if available, determines whether one fresh artifact can now be produced, and maps the model-call entrypoints for a later Artifact Cost Guardrails v1 slice.

Primary questions:
1. Can the local DAI factory now generate at least one fresh sports decision artifact end-to-end?
2. Which protocols/stations leverage model calls directly or indirectly during sports artifact generation?

## Scope

- read the prior handoff and operability-fix report
- start/verify FastAPI analyzer, .NET platform API, Docker, and `devcore-sql`
- rerun the direct FastAPI analyze smoke and the full `.NET /api/agent-runs` smoke
- a very small structural sanity check of any fresh artifact
- a read-only inventory of model-call entrypoints and protocol/station relationships

## Non-goals

- no prompt or buyer-copy change
- no confidence/posture/lean/signal/decision-logic change
- no source addition, no probe-refresh activation
- no schema change
- no cost guardrails, metering, or pricing implementation (this slice only inventories where spend enters)
- no auth/billing/tenant/dashboard/Kubernetes/deployment work
- no Jera work, no push

## Prior blockers

1. OpenAI quota exhausted -- direct analyze returned `429 insufficient_quota`. User has now added credits.
2. `devcore-sql` not running -- full chain failed at `IdentityResolver.ResolveDevelopmentAsync` with a SQL connection timeout.

## OpenAI credit blocker result

Cleared. Direct analyze now returns HTTP 200 with a real model response.

- command: `POST http://127.0.0.1:8000/api/sports/analyze` with an NBA matchup (`Boston Celtics` vs `Denver Nuggets`, `2026-06-12`), header `X-Agent-Run-Id: postcredit-nba-0609`.
- result: HTTP 200 in ~9.6s. Returned a full `protocol` object (perceive; interrogate.question/verify; discern.contrast/weigh/stress; decide.justify/position/resolve), `lean`, `summary`, `confidence`, `posture`, `counter_case`, `watch_for`, `what_would_change_the_read`.
- no `429`, no `insufficient_quota`. The credit issue is fixed.

## Docker / SQL blocker result

Cleared in this session. Docker Desktop was started; the daemon came up (server 29.1.3). The `devcore-sql` container existed but was `Exited (137)` from 8 days ago. `docker start devcore-sql` brought it up publishing `0.0.0.0:1433->1433`; the container log shows "SQL Server is now ready for client connections" at the current time.

Note: this required manually starting Docker Desktop and the container. Both are local environment actions, not code.

## Services started and health checks

| service | start command | bound | health |
|---|---|---|---|
| FastAPI analyzer | `.\.venv\Scripts\python.exe -m uvicorn main:app --host 127.0.0.1 --port 8000` | 127.0.0.1:8000 | `GET /api/ping` -> 200 `{"status":"ok"}` |
| .NET platform API | `ASPNETCORE_ENVIRONMENT=Development dotnet run --project .\DevCore.Api\DevCore.Api.csproj` | localhost:5007 | `GET /health` -> 200 `{"status":"ok"}` |
| Docker daemon | `Start-Process "Docker Desktop.exe"` | npipe | `docker info` -> server 29.1.3 |
| devcore-sql | `docker start devcore-sql` | 0.0.0.0:1433 | container log "ready for client connections" |

## Direct FastAPI analyze result

HTTP 200, model call succeeded (see OpenAI credit blocker result). The single analyzer call emits the entire cognitive `protocol` block. There was no `probe` and no `synthesize` field in the analyzer's own output -- those are platform-completed by .NET, not model-emitted.

## Full .NET chain result

Works end-to-end.

- command: `POST http://localhost:5007/api/agent-runs` with `runType: sports.matchup.analysis` and the NBA input above.
- result: HTTP 200 in ~10.1s. `status: "completed"`, `agentRunId: 946b433e-f36b-1410-8161-00373db4b724`, `durationMs: 8750`, `posture: "wait"`, `confidence: 0.375` (analyzer-local 0.5 was recomputed deterministically by the .NET evaluate step).
- the full Angular -> .NET -> FastAPI -> SQL path now manufactures and persists a fresh artifact.

## Artifact generated

Yes.

- artifact version: `sports_decision_artifact_v3`
- league: NBA (`Boston Celtics` vs `Denver Nuggets`, `2026-06-12`)
- `cognitiveProtocol`: present (canonical block persisted; `protocolView` also surfaced)
- buyer-facing projection: available (`lean`, `summary`, `posture`, `counterCase`, `watchFor`, `whatWouldChangeTheRead`)
- pipeline steps: retrieve (Degraded -- "no signals grounded for nba -- priors-only run"), analyze (Succeeded), evaluate (Succeeded), quality_check (Succeeded), compose (Succeeded)
- output location: persisted to the dev `devcore` SQL database as `AgentRun` row `946b433e-f36b-1410-8161-00373db4b724`, retrievable via `GET /api/agent-runs/{id}/artifact`. Not saved to the repo: this was a raw smoke, not a calibration-harness batch, so it was deliberately not committed as a sample (committing convention-named artifact samples belongs to `run-artifact-calibration.ps1` in the dedicated calibration slice).

Note: this was a priors-only run (no live signals grounded for that future date/matchup), so the read is intentionally `No clear lean` with `wait` posture. That is correct behavior for a thin-signal run; signal richness is not this slice's concern.

## Buyer artifact sanity check

Pass (small structural check only, not a quality pass).

- structurally complete: all expected buyer and diagnostic fields present; five pipeline steps recorded.
- `cognitiveProtocol` exists and `artifactVersion` is v3.
- buyer projection is available and coherent.
- unsafe/tout language scan: clean. The only regex hit (`lock`) was a substring of the internal fields `block_aggressive_posture` and `aggressive_blocked`; there is no standalone tout term, and those fields are internal diagnostics, not buyer copy.

## Protocol model-call inventory (read-only)

| Area | Model call? | How known | Notes |
|---|---|---|---|
| Perceive | Indirect | `services/agent-service/app/services/sports_analyzer.py:477` `_call_model` (single call); live smoke `protocol.perceive` | content produced by the one analyzer model call |
| Interrogate.Question | Indirect | same single call; `protocol.interrogate.question` | same call |
| Interrogate.Probe | No | not in FastAPI model output; `.NET` `protocolView.interrogate.probe` is platform-completed; probe-refresh dormant (ledger entries 1-2) | deterministic/platform-completed; `ProbeRefreshExecutor` disabled |
| Interrogate.Verify | Indirect | same single call; `protocol.interrogate.verify` | same call |
| Discern | Indirect | same single call; `protocol.discern.contrast/weigh/stress` | same call |
| Decide | Indirect | same single call emits `decide.justify/position/resolve`; final confidence is recomputed deterministically by the .NET evaluate step and posture is clamped | model proposes; .NET owns final confidence (`SportsEvaluator`) and posture clamp (0.5 analyzer -> 0.375 final) |
| Synthesize | No | `.NET`; `protocolView.synthesize` is constant "platform operation: ..." text | deterministic platform projection, no model |
| Buyer projection/UI | No | `.NET` compose step + Angular packaging | deterministic, no model |

The architecture does map cleanly: model usage is centralized at the analyzer level (one call per run) and only later projected into protocol structure by .NET. No per-station model calls exist.

## Model-call entrypoints

- Sports path: exactly one model call per run, at `sports_analyzer._call_model` (`sports_analyzer.py:480`, `chat.completions.create`, `gpt-4o-mini`, `temperature 0.3`, `response_format json_object`). Reached through FastAPI `POST /api/sports/analyze`, dispatched by `analyze_football` / `analyze_basketball` / `analyze_mlb`.
- Non-sports path (not part of artifact generation): a separate model call in `main.py:113` (`gpt-4o-mini`) powers the generic `/api/chat`, `/api/assist`, and gRPC streaming endpoints. It is not invoked during sports artifact generation.

## Does .NET call models directly?

No. `.NET` (`DevCore.AiClient.FastApiClient.AnalyzeSportsMatchupAsync`) only HTTP-POSTs to FastAPI `/api/sports/analyze` and forwards `X-Agent-Run-Id` + `X-Correlation-Id`. All other `.NET` HTTP clients (`OddsScheduleClient`, `OddsMarketClient`, `MlbStarterClient`, `EspnBasketballScheduleClient`, `ActionNetworkClient`) are deterministic signal providers, not model calls. The model boundary is entirely inside FastAPI.

## Deterministic-only areas

- Synthesize (integrate/compose/deliver) -- platform-operational constant text.
- Buyer projection / UI packaging.
- Interrogate.Probe -- platform-completed.
- Final confidence (`SportsEvaluator`) and posture clamp -- deterministic in .NET.
- All `.NET` signal providers (odds, schedule, starters, sharp/public).

## Dormant / non-active paths

- The probe-refresh chain is dormant and non-model-calling today: `ProbeRefreshExecutorOptions.Enabled = false`, no merge writer, recommendation-only Decide (deferred-runtime-decisions ledger entries 1-5). None of it makes model calls.

## Current observability of token / cost / latency

Effectively none at the model boundary.

- the analyzer never reads `response.usage`; prompt/completion/total token counts are not captured. The only model-related log is `log.info("%s analysis response: %d chars", ...)` -- response character count only.
- no per-call logging of model name, request id, latency, retry count, or cost in the analyzer.
- no `max_tokens` cap, no explicit per-call timeout, and no retry/backoff on the model call.
- `.NET` logs only FastAPI non-2xx status + a truncated error body (`FastApiClient.cs:32`); it captures whole-run `durationMs` (e.g. 8750ms) at the agent-run level, which is pipeline duration, not isolated model latency.
- model name `gpt-4o-mini` is hardcoded in two places (`sports_analyzer.py:481`, `main.py:114`); there is no central model config.
- `.NET` FastApiClient HttpClient timeout is 90s (`Program.cs:67`); deterministic providers use 10-15s.

## Gaps to address later in Artifact Cost Guardrails v1

- capture `response.usage` (prompt/completion/total tokens) at `_call_model` and attach to the run.
- structured per-call logging: model name, request id (already have `X-Agent-Run-Id`), latency, token counts, retry count, estimated cost.
- central model config (name, max_tokens, timeout, retry/backoff) instead of two hardcoded `gpt-4o-mini` literals.
- a metering/audit sink and (later, separate slice) caps. The single centralized model site makes instrumentation low-risk.

This inventory informs, but does not implement, Artifact Cost Guardrails v1.

## Remaining blockers

None for fresh generation. The full chain works with credits added and `devcore-sql` running. The only operational caveat is that Docker Desktop + `devcore-sql` must be running locally; the container persists in `Exited` state across reboots and is restarted with `docker start devcore-sql`.

## Recommended next slice

Two candidates, in priority order:
1. Artifact Cost Guardrails v1 -- instrument the single analyzer model call (token usage, cost, latency, model name, retry) and add a metering/audit sink before any caps or pricing. Inventory above is the input.
2. Fresh Buyer Artifact Generation Calibration v1 -- now that the line is proven, run a small fresh batch via `run-artifact-calibration.ps1` over real upcoming games to revisit the quality-watch items from the prior quality pass (MLB single-signal high-confidence, phrase repetition) with genuinely fresh post-safety artifacts.

## Files changed

- `dai-vault/04 Products/sports-v1/post-credit-fresh-artifact-generation-smoke-v1.md` (new -- this report)
- `dai-vault/06 Execution/handoffs/current-slice.md` (addendum)
- `dai/scripts/dev/sports/README.md` (one line: explicit `docker start devcore-sql` in the SQL troubleshooting)

## Deferred decisions / ledger

No ledger update needed. No new deferred runtime/prompt/source/artifact-contract decision was discovered. This slice confirms existing deferrals (probe-refresh dormant, entries 1-5) rather than creating new ones. Cost-guardrail design is a recommended future slice, not yet a consciously deferred runtime decision with a committed seam.

## Verification performed

- repo baseline for `dai`, `dai-vault` (both clean, even with origin); jera not present.
- FastAPI: `GET /api/ping` -> 200; `POST /api/sports/analyze` -> 200 (429 cleared).
- Docker: daemon up (server 29.1.3); `devcore-sql` started, publishing 1433, "ready for client connections".
- .NET: `GET /health` -> 200; `POST /api/agent-runs` -> 200 `completed`.
- artifact: `artifactVersion: sports_decision_artifact_v3`, `cognitiveProtocol` present, buyer projection present, unsafe-language scan clean.
- docs-only verification for the README/vault changes: `git diff --check`, added-line exact-path scan, vault non-ASCII added-line scan.

## What was not changed

- no FastAPI prompt, route, or model code
- no parser / sanitation behavior
- no .NET runtime logic, controller, or composer
- no confidence/posture/lean/decision/artifact-semantics change
- no schema or migration
- no source expansion, no probe-refresh activation
- no Angular code
- no cost-guardrail, metering, or pricing implementation
- no auth/billing/tenant/dashboard/Kubernetes/deployment work
- no Jera work
- no `OPENAI_API_KEY` value change and no secret written to the repo
