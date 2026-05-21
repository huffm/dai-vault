# cloud deploy readiness v1

**date:** 2026-05-21
**status:** readiness assessment. no deployment performed. no runtime code changed.
**scope:** what is needed before a real Azure Container Apps deploy of the sports runtime, and what is deliberately deferred.

## what this document is

The sports runtime now routes reference, retrieve, and analyze calls through the Tool Gateway with telemetry (see `cloud-tool-runtime-plan.md` and the Tool Gateway slices in `06 Execution/handoffs/current-slice.md`). This document inventories the deployable services as they exist today, names the target Container Apps shape, and lists the concrete blockers between here and a first cloud deploy. It is launch-forward and intentionally small: it does not author Dockerfiles, change config, or move secrets. It tells the next slice exactly what to build.

Container Apps is the launch target. AKS, MCP, pgvector, and Azure Functions stay deferred.

## 1. service inventory

| service | code location | role | local port | container app name (proposed) |
|---|---|---|---|---|
| .NET API | `dai/platform/dotnet/DevCore.Api` | orchestrator: pipeline, Tool Gateway, identity, persistence | 5007 | `dai-api` |
| FastAPI analyzer | `dai/services/agent-service` | the single sports analysis model call | 8000 (http) + 50051 (grpc) | `dai-analyzer` |
| Angular sports app | `dai/apps/sports-app` | customer-facing web client (static) | 4201 | `dai-sports-web` |
| SQL Server | `devcore-sql` docker container locally | system of record (`AgentRun`, `Sports`, `Teams`, ...) | 1433 | Azure SQL (managed, not a container app) |

The proposed container app names use a `dai-` product prefix with a role suffix (`-api`, `-analyzer`, `-sports-web`). They are lowercase, hyphenated, stable, and map cleanly to the code: `dai-api` = `DevCore.Api`, `dai-analyzer` = the FastAPI agent-service analyzer, `dai-sports-web` = the sports-app. The assembly name stays `DevCore.Api`; only the cloud resource is `dai-api`.

## 2. current local runtime shape

- `scripts/dev/sports/start-sports-dev.ps1` launches three processes: the FastAPI analyzer (`uvicorn main:app --host 127.0.0.1 --port 8000`), the .NET API (`dotnet run`, port 5007), and the Angular app (port 4201).
- SQL Server runs as the `devcore-sql` docker container on port 1433. It is not a Windows service. The .NET API connects via `ConnectionStrings:Sql`.
- The .NET API reaches the analyzer at `AiService:BaseUrl = http://127.0.0.1:8000` (set in `appsettings.Development.json`). Note: an earlier draft of `cloud-tool-runtime-plan.md` referenced port 8001; the actual analyzer port is 8000. Treat 8000 as truth.
- FastAPI also starts a gRPC server on port 50051 in its lifespan. The sports analysis path does not use gRPC; `FastApiClient` calls `POST /api/sports/analyze` over HTTP only. gRPC is out of scope for the sports launch.
- The Angular client targets `apiBaseUrl: http://localhost:5007` in both `environment.ts` and `environment.development.ts`.
- No Dockerfiles, no `.dockerignore`, and no compose file exist in the repo today. All three services run from source.
- The .NET API has no health endpoint. The FastAPI analyzer has `GET /api/ping` (used by `test-sports-dev.ps1`).

## 3. target azure container apps shape

Three container apps in one Container Apps environment, one region:

- `dai-api`: public (external) ingress on the HTTPS edge. Holds the Tool Gateway, identity, and persistence. Talks to `dai-analyzer` over the environment's internal network.
- `dai-analyzer`: internal ingress only. Reachable from `dai-api` at the internal DNS name `http://dai-analyzer` (the Container Apps internal FQDN). Not publicly exposed. gRPC port 50051 stays unexposed.
- `dai-sports-web`: served as a static site (Azure Static Web Apps or Storage static site behind a CDN). It is not a long-running container; it is a build artifact. It calls the public `dai-api` URL.
- Azure SQL (managed) as the system of record. No schema change from local.

This matches `cloud-tool-runtime-plan.md` section 1. No broker, no API Management, no AKS.

## 4. required env vars

`dai-api` (.NET, uses the standard `Section__Key` double-underscore convention):

| env var | purpose | source |
|---|---|---|
| `ASPNETCORE_ENVIRONMENT` | `Production` in cloud | platform |
| `ConnectionStrings__Sql` | Azure SQL connection string | Key Vault |
| `AiService__BaseUrl` | internal analyzer URL, e.g. `http://dai-analyzer` | platform config |
| `AzureAd__TenantId` | Entra tenant id (not a secret) | platform config |
| `AzureAd__Audience` | api audience (not a secret) | platform config |
| `OddsApi__ApiKey` | odds api key | Key Vault |
| `Dev__EnableBypassAuth` | must be `false` in cloud | platform config |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | telemetry sink | Key Vault or platform config |

`dai-analyzer` (FastAPI, `os.getenv`):

| env var | purpose | source |
|---|---|---|
| `OPENAI_API_KEY` | model provider key | Key Vault |
| `UVICORN_HOST` / port | bind `0.0.0.0` and the container port | container CMD |

`dai-sports-web` (Angular): `apiBaseUrl` is a build-time value baked into `environment.ts`, not a runtime env var. The production build must set it to the public `dai-api` URL before the static build. This is a build-pipeline input, not a container env var.

## 5. secrets and key vault plan

Move to Key Vault, referenced from Container Apps as secret refs:

- `ConnectionStrings__Sql` (Azure SQL)
- `OddsApi__ApiKey`
- `OPENAI_API_KEY`
- `APPLICATIONINSIGHTS_CONNECTION_STRING` (if treated as a secret)

Not secrets (plain config): `AzureAd__TenantId`, `AzureAd__Audience`, `AiService__BaseUrl`, `ASPNETCORE_ENVIRONMENT`, `Dev__EnableBypassAuth`.

**Urgent finding (security): secrets are committed to the repo today.**

- `appsettings.json` and `appsettings.Development.json` both contain a real SQL Server password in `ConnectionStrings:Sql`.
- `appsettings.Development.json` contains a real `OddsApi:ApiKey` value (despite an inline comment saying to use user secrets) and a `Dev:ProvisionKey` test value.

These must be rotated and removed from source before any cloud work, and ideally before the next commit. The values are now in git history, so rotation (not just deletion) is required. This is the top blocker and is called out again in section 11. This readiness slice does not edit those files; it flags them.

## 6. health check plan

- `dai-api`: no health endpoint exists. Add a minimal liveness endpoint (for example `GET /health` returning 200) so Container Apps can probe it. This is a small code change for the containerize slice, not done here.
- `dai-analyzer`: reuse the existing `GET /api/ping` as the liveness probe. No new code needed.
- `dai-sports-web`: static host serves `index.html`; the platform's static-site health is sufficient. No app probe.
- Azure SQL: managed; platform-owned health.

## 7. internal networking plan

- `dai-api` is the only public ingress. `dai-analyzer` is internal-only.
- `dai-api -> dai-analyzer` over the Container Apps internal network. Set `AiService__BaseUrl = http://dai-analyzer`. The wire contract (`POST /api/sports/analyze` with `X-Correlation-Id` and `X-Agent-Run-Id` headers) is unchanged from local; only the hostname changes from `127.0.0.1:8000` to the internal DNS name.
- `dai-sports-web -> dai-api` over the public edge. CORS in `dai-api` must allow the deployed web origin (today it allows localhost dev origins only).
- gRPC (50051) is not exposed and not used by the sports path.

## 8. logging and telemetry plan

- The Tool Gateway already emits one structured `ToolGatewayInvocation` log per tool call (tool id, protocol node, run/tenant/correlation ids, cost class, outcome, duration). This is the observability surface for cloud.
- Wire Application Insights into both `dai-api` and `dai-analyzer` via `APPLICATIONINSIGHTS_CONNECTION_STRING`. Correlate on `X-Agent-Run-Id` (canonical) and `X-Correlation-Id`, which already flow on the internal analyze call.
- No new telemetry abstraction is needed; the gateway structured logs plus App Insights ingestion are enough for launch.

## 9. what is launch-required

- Dockerfiles for `dai-api` (multi-stage .NET) and `dai-analyzer` (python slim + uvicorn). A static build for `dai-sports-web`.
- All committed secrets rotated and externalized to env vars / Key Vault.
- `Dev__EnableBypassAuth = false` in cloud, with real customer auth (Entra External ID) in place before public exposure. See `next-platform-architecture-plan.md`.
- A `/health` endpoint on `dai-api`.
- `dai-sports-web` production `apiBaseUrl` pointed at the deployed `dai-api` URL.
- `dai-api` CORS updated to allow the deployed web origin.
- A managed Azure SQL instance and its connection string in Key Vault.

## 10. what is post-launch

- pgvector memory, Azure Functions (reflex jobs), MCP adapters, AKS, multi-region, API Management. All deferred per `cloud-tool-runtime-plan.md`.
- Exposing or hosting the FastAPI gRPC listener.
- Gateway idempotency-cache and cost-class enforcement (registry fields are declarative today).
- Tenant-tier enforcement on tool calls.

## 11. deployment blockers

1. **Committed secrets (security, highest priority).** SQL password in both appsettings files; OddsApi key and Dev provision key in `appsettings.Development.json`. Rotate and remove before cloud work.
2. **No Dockerfiles.** All three services run from source today; none is container-ready.
3. **Auth bypass on.** `Dev:EnableBypassAuth = true` in dev. Cloud must be `false`, and real customer auth must exist before the public API is exposed.
4. **No `dai-api` health endpoint.** Container Apps needs a liveness probe target.
5. **Angular API URL hardcoded to localhost.** Production `environment.ts` must point at the deployed `dai-api`.
6. **No managed SQL provisioned.** Local points at the `devcore-sql` docker container; cloud needs Azure SQL plus the connection string in Key Vault.
7. **CORS allows dev origins only.** Must add the deployed web origin.

Blockers 2, 4, 5, and 7 are small build/config tasks. Blockers 1, 3, and 6 are gating and partly outside a single code slice (secret rotation, auth, infra provisioning).

## 12. recommended next slice

Two ordered slices:

1. **Secrets hygiene (do first, urgent).** Rotate the committed SQL password and OddsApi key, remove real values from `appsettings*.json`, move local dev to user secrets / a gitignored `.env`, and confirm nothing secret remains tracked. This is a prerequisite for any cloud work and reduces current exposure regardless of deploy timing.
2. **Containerize (the build slice).** Author the three Dockerfiles, add the `dai-api` `/health` endpoint, externalize config to env vars (no behavior change locally because `appsettings.Development.json` still supplies dev values), and verify a local container build of each service plus parity with `test-sports-dev.ps1`. No Azure resources created yet; this proves the images build and run before any cloud provisioning.

The actual Container Apps provisioning (environment, internal DNS, Key Vault wiring, App Insights, public ingress) follows once images build cleanly and secrets are out of the repo.

## references

- `02 Platform/architecture/cloud-tool-runtime-plan.md`
- `02 Platform/architecture/next-platform-architecture-plan.md`
- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
- `06 Execution/handoffs/current-slice.md`
