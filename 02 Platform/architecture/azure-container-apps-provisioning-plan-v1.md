# azure container apps provisioning plan v1

**date:** 2026-05-22
**status:** provisioning plan. no Azure resources created in this slice. docs only.
**scope:** the precise, named plan to provision the sports runtime on Azure Container Apps. it tells the provisioning slice exactly what to create, what to name it, and in what order, without taking deploy risk yet.

## what this document is

Containerization Readiness v1 and Local Container Compose Smoke v1 are complete: `dai-api` and `dai-analyzer` build as containers, `dai-api` serves `GET /health`, `dai-analyzer` serves `GET /api/ping`, and the two reach each other by service name on a private network (proven by `compose.smoke.yaml`). This document is the next step: a concrete Azure Container Apps provisioning plan. It names every resource, fixes ingress and networking, lists env vars and Key Vault secrets, and sequences build, deploy, smoke, and rollback. It does not create anything in Azure.

This plan is bounded by the launch surface in `cloud-tool-runtime-plan.md` and the readiness assessment in `cloud-deploy-readiness-v1.md`. Container Apps is the launch host. AKS, MCP, pgvector, Azure Functions, and multi-region stay deferred (section 19).

Architecture guardrails this plan preserves: the Cognitive Protocol Runtime behavior, Tool Gateway governance, CognitiveProtocol persistence, and the local dev workflow are all unchanged. Nothing here bypasses the Tool Gateway; the published images already contain the gateway DI, telemetry, and the analyze wrap. No FastAPI prompt, Pydantic contract, `CognitiveProtocolBuilder` mapping, confidence rule, database schema, Angular behavior, MCP, pgvector, or Azure Function is touched.

## 1. azure resource naming plan

Naming follows the Azure Cloud Adoption Framework abbreviation convention (`rg-`, `cr`, `cae-`, `kv-`, `log-`, `appi-`, `sql-`, `swa-`, `id-`) with a `dai` workload token and a `prod` environment token, except for the three Container Apps themselves, which keep the doctrine-locked names `dai-api`, `dai-analyzer`, `dai-sports-web` (see the naming review note below).

| resource | type | proposed name | constraint notes |
|---|---|---|---|
| resource group | Resource Group | `rg-dai-prod` | one region, one workload |
| container registry | Azure Container Registry | `crdaiprod` | globally unique, 5-50 alphanumeric, no hyphens allowed |
| container apps environment | Container Apps Environment | `cae-dai-prod` | one environment hosts all three apps |
| .NET orchestrator | Container App | `dai-api` | doctrine-locked, public ingress |
| FastAPI analyzer | Container App | `dai-analyzer` | doctrine-locked, internal ingress only |
| Angular client | Static Web App | `swa-dai-sports-web` | static hosting, not a container app (see section 6) |
| key vault | Key Vault | `kv-dai-prod` | globally unique, 3-24 chars, alphanumeric and hyphens |
| log analytics workspace | Log Analytics Workspace | `log-dai-prod` | backs the Container Apps environment and App Insights |
| application insights | Application Insights (workspace-based) | `appi-dai-prod` | telemetry sink for both containers |
| azure sql server | SQL logical server | `sql-dai-prod` | globally unique hostname `sql-dai-prod.database.windows.net` |
| azure sql database | SQL database | `devcore` | keep the local database name to avoid connection-string drift |
| managed identity (api) | User-assigned managed identity | `id-dai-api` | optional; pulls ACR images and reads Key Vault (section 11) |

**naming review result.** The three Container Apps deliberately do not take the CAF `ca-` prefix. Their names are already locked across `compose.smoke.yaml`, both Dockerfiles, and `cloud-deploy-readiness-v1.md`, and the internal service URL contract depends on the app name: `dai-api` reaches the analyzer at `http://dai-analyzer` because the app is named `dai-analyzer`. Renaming to `ca-dai-analyzer` would force a config change and break the documented `AiService__BaseUrl` value for no benefit. Every other resource follows CAF abbreviations. ACR drops hyphens because the registry name forbids them. The Static Web App takes `swa-` because it is genuinely a different resource type, not a container app. Names reviewed: resource group, ACR, ACR image tags, Container Apps environment, the three container apps, Static Web App, Key Vault, Key Vault secret names (section 11 wait, see section 10/11), env vars (section 9), ingress targets (section 7), Azure SQL server and database, managed identity. All are lowercase, hyphenated where the resource type allows, product-prefixed, stable, and boring. No misleading names found.

## 2. resource group name

`rg-dai-prod`. One resource group holds the full launch footprint in one region: the Container Apps environment and its three apps, the container registry, Key Vault, the Log Analytics workspace and Application Insights, the Azure SQL server and database, and the Static Web App. A single group keeps the launch deployment, RBAC, and teardown atomic. A second group is not warranted at this footprint.

## 3. region recommendation and open question

**Recommendation:** `eastus2`. It carries Container Apps, Azure SQL, Static Web Apps, Application Insights, and (for the deferred pgvector slice) Azure Database for PostgreSQL Flexible Server in the same region, has broad capacity, and is a stable default for a US-targeted sports product.

**Open question (carry forward, decide before provisioning):** the customer base and latency profile have not been fixed, and Entra External ID tenant region is a separate choice from the compute region. Pick the region against (a) where the first paying customers are, (b) Azure SQL and Static Web Apps availability in that region, and (c) the Entra External ID tenant data residency. If there is no signal, `eastus2` is the default and every resource in section 1 colocates there. Keep all resources in one region for launch; multi-region is deferred (section 19).

## 4. azure container registry image names

Registry `crdaiprod` (login server `crdaiprod.azurecr.io`). Two images; the Angular client is a static build with no image.

| image | repository | source Dockerfile | build context |
|---|---|---|---|
| dai-api | `crdaiprod.azurecr.io/dai-api` | `platform/dotnet/Dockerfile` | `dai/platform/dotnet` |
| dai-analyzer | `crdaiprod.azurecr.io/dai-analyzer` | `services/agent-service/Dockerfile` | `dai/services/agent-service` |

Tagging: tag each push with both an immutable build id and a moving channel tag. Use the git short sha as the immutable tag (`dai-api:<gitsha>`) and `:prod` as the channel pointer for the current released revision. Container Apps revisions reference the immutable sha tag so a rollback is a revision pin, not a tag chase (section 17). Do not deploy `:latest`.

## 5. container apps environment name

`cae-dai-prod`. One Container Apps environment backed by the `log-dai-prod` Log Analytics workspace. All three apps run inside it so that internal ingress and internal DNS work between `dai-api` and `dai-analyzer` without any public exposure of the analyzer. The Static Web App (`swa-dai-sports-web`) is not inside the environment; it is a separate static-hosting resource that calls the public `dai-api` URL.

## 6. container app names

- **`dai-api`** (the `DevCore.Api` orchestrator). Public external ingress. Holds the pipeline, the Tool Gateway, identity, persistence, and CORS. Container listens on 8080 (`ASPNETCORE_URLS=http://+:8080`, baked in the Dockerfile).
- **`dai-analyzer`** (the FastAPI agent-service). Internal ingress only. Container listens on 8000 (uvicorn). The gRPC listener on 50051 stays unexposed and unused on the sports path.
- **`dai-sports-web`** (the Angular sports-app), provisioned as **Azure Static Web Apps** named `swa-dai-sports-web`. This is the chosen static-hosting alternative to running the client as a container app. Rationale: the Angular client is a build artifact that holds no secrets and needs no server runtime; Static Web Apps gives free TLS, global CDN, and a deploy pipeline without a long-running container. Storage static site behind a CDN is the fallback if Static Web Apps constraints get in the way, but Static Web Apps is the recommendation. Its production `apiBaseUrl` is baked at build time (section 13), not a runtime env var.

## 7. ingress rules

- **`dai-api`: public (external) ingress.** This is the only public edge. HTTPS on the Container Apps managed FQDN (`dai-api.<env-id>.<region>.azurecontainerapps.io`) at launch; a custom domain can be added later. Target port 8080. Allow insecure traffic = false (HTTPS only). This app must not be publicly reachable with `Dev__EnableBypassAuth=true`; see section 14.
- **`dai-analyzer`: internal ingress only.** Ingress type internal, target port 8000. Reachable only from inside `cae-dai-prod` (that is, from `dai-api`). No public FQDN. The model-call analyzer is never exposed to the internet. gRPC 50051 is not added to ingress.
- **`dai-sports-web`** is served by Static Web Apps over its own HTTPS edge; it is not part of Container Apps ingress. It calls the public `dai-api` URL from the browser (CORS, section 13).

## 8. internal url for AiService__BaseUrl

With `dai-analyzer` on internal ingress in `cae-dai-prod`, set on `dai-api`:

```
AiService__BaseUrl = http://dai-analyzer
```

Container Apps internal ingress exposes the app at its app name on port 80 and routes to the container target port (8000). So the value is `http://dai-analyzer` with no port, not `http://dai-analyzer:8000`. This differs from `compose.smoke.yaml`, which uses `http://dai-analyzer:8000` because Compose has no ingress layer and hits the container port directly. The wire contract is unchanged: `POST /api/sports/analyze` with `X-Correlation-Id` and `X-Agent-Run-Id`; only the hostname changes from `127.0.0.1:8000` (local) to `dai-analyzer` (cloud). If short-name resolution is unreliable in a given environment, the fully qualified internal FQDN `http://dai-analyzer.internal.<env-id>.<region>.azurecontainerapps.io` is the explicit fallback.

## 9. required environment variables

**`dai-api`** (.NET, `Section__Key` double-underscore convention):

| env var | value / source | secret? |
|---|---|---|
| `ASPNETCORE_ENVIRONMENT` | `Production` | no |
| `ASPNETCORE_URLS` | `http://+:8080` (baked in Dockerfile; keep explicit) | no |
| `ConnectionStrings__Sql` | Azure SQL connection string | yes, Key Vault ref |
| `AiService__BaseUrl` | `http://dai-analyzer` (section 8) | no |
| `AzureAd__TenantId` | Entra External ID tenant id | no |
| `AzureAd__Audience` | api app registration audience (`api://<api-app-id>`) | no |
| `OddsApi__ApiKey` | odds api key | yes, Key Vault ref |
| `Dev__EnableBypassAuth` | `false` (must be false in cloud) | no |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | App Insights connection string | yes (treat as secret), Key Vault ref |

**`dai-analyzer`** (FastAPI, `os.getenv`):

| env var | value / source | secret? |
|---|---|---|
| `OPENAI_API_KEY` | model provider key | yes, Key Vault ref |
| host/port | `0.0.0.0:8000` (baked in Dockerfile CMD) | no |

**`dai-sports-web`** (Static Web Apps): `apiBaseUrl` is a build-time value in `environment.ts`, not a runtime env var (section 13).

## 10. key vault secret names

Key Vault `kv-dai-prod`. Secrets are referenced from the container apps as secret refs (Key Vault references resolved by the app's managed identity). Key Vault secret names cannot contain underscores, so the double-underscore env keys map to hyphenated secret names; the Container App secret definition maps the hyphenated vault secret to the underscore env var.

| key vault secret name | maps to env var | consumed by |
|---|---|---|
| `connectionstrings-sql` | `ConnectionStrings__Sql` | dai-api |
| `oddsapi-apikey` | `OddsApi__ApiKey` | dai-api |
| `appinsights-connection-string` | `APPLICATIONINSIGHTS_CONNECTION_STRING` | dai-api (and dai-analyzer if instrumented) |
| `openai-api-key` | `OPENAI_API_KEY` | dai-analyzer |

Non-secret config stays as plain env values, not Key Vault: `AzureAd__TenantId`, `AzureAd__Audience`, `AiService__BaseUrl`, `ASPNETCORE_ENVIRONMENT`, `Dev__EnableBypassAuth`. Per the Tool Gateway doctrine, no secret ever lands on the artifact, in `OutputJson`, or in a log line; the gateway reads provider keys from configuration injected by Key Vault refs, not from the run context.

## 11. app insights / log analytics plan

- Provision `log-dai-prod` (Log Analytics workspace) first; it backs both the Container Apps environment diagnostics and the workspace-based Application Insights resource.
- Provision `appi-dai-prod` (workspace-based App Insights) against `log-dai-prod`.
- Wire `APPLICATIONINSIGHTS_CONNECTION_STRING` into `dai-api` (and optionally `dai-analyzer`) as a Key Vault ref.
- The observability surface already exists: the Tool Gateway emits one structured `ToolGatewayInvocation` log per tool call (tool id, protocol node, run id, tenant key, correlation id, cost class, outcome, duration_ms). In cloud these surface as App Insights traces with custom dimensions. Correlate on `X-Agent-Run-Id` (canonical) and `X-Correlation-Id`, which already flow on the internal analyze call. No new telemetry abstraction is needed for launch.
- Identity for Key Vault: give each container app a managed identity (system-assigned is simplest; the user-assigned `id-dai-api` is an option if the same identity should also pull ACR images) and grant it `get`/`list` on Key Vault secrets via RBAC or access policy.

## 12. azure sql plan and connection string source

- Provision `sql-dai-prod` (SQL logical server) and database `devcore` in `rg-dai-prod`, same region. Keep the database name `devcore` so the connection string differs from local only in server host and credentials.
- No schema change from local. The runtime persists `AgentRun`, `Sports`, `Teams`, `Users`, `UserIdentity` exactly as today; CognitiveProtocol persistence in `OutputJson` is unchanged.
- Connection string source: the full Azure SQL connection string is stored as the Key Vault secret `connectionstrings-sql` and injected into `dai-api` as `ConnectionStrings__Sql` (section 10). The baked placeholder in `appsettings.json` (`__REPLACE_VIA_USER_SECRETS_OR_ENV__`) is never used in cloud because the env var overrides it.
- Connectivity: allow the Container Apps environment outbound to Azure SQL (Azure SQL "allow Azure services" or, preferred, the environment's outbound IPs / a private endpoint). Decide the network path in the provisioning slice; a private endpoint is the more secure default if the environment supports it cleanly.
- Schema bootstrap: confirm whether the production database is created by EF migrations on first run or by an explicit migration step. This is a provisioning-slice decision; the schema itself is not changed here.

## 13. cors plan

- CORS is currently hardcoded in `DevCore.Api/Program.cs:22-26` as the `spa` policy with dev origins only (`https://localhost:4200`, `http://localhost:4201`, and the 127.0.0.1 variants). It is applied at `Program.cs:142` (`app.UseCors("spa")`).
- For cloud, `dai-api` must allow the deployed `swa-dai-sports-web` origin (its Static Web Apps HTTPS URL, and any custom domain later). The cleanest path is to externalize the allowed origins to configuration (for example a `Cors:AllowedOrigins` array read in `Program.cs`) so the production origin is an env var, not a code constant. That externalization is a small code change and belongs to the provisioning/build slice, not this docs slice; this plan flags it as required.
- Until externalized, the alternative is to add the production origin directly in the `spa` policy. Externalizing is preferred so origins differ by environment without a rebuild.
- Set the Angular `apiBaseUrl` (in `environment.ts`, currently `http://localhost:5007`) to the public `dai-api` HTTPS URL at build time before the Static Web Apps build. The browser then calls `dai-api` cross-origin and `dai-api` CORS must name that origin. The two must agree.

## 14. customer auth / entra external id gating

- `dai-api` already wires JWT bearer auth in `Program.cs:42-53` with authority `https://login.microsoftonline.com/{AzureAd:TenantId}/v2.0` and audience `AzureAd:Audience`. This is the workforce Entra authority shape. **Customer identity uses Entra External ID (CIAM), whose authority host is different** (the `ciamlogin.com` form), so moving to customer auth means setting `AzureAd__TenantId`/`AzureAd__Audience` to the External ID tenant and app registration, and confirming the authority host the code builds is correct for External ID. Whether the authority string in `Program.cs:47` needs to change for External ID is a decision for the auth slice; this plan flags it, it is not changed here.
- `Dev__EnableBypassAuth` must be `false` in cloud. In dev, `IdentityResolver.cs:40` honors the bypass only when `env.IsDevelopment()` and the flag is true; in `Production` with the flag false, real bearer auth is enforced. Setting `ASPNETCORE_ENVIRONMENT=Production` and `Dev__EnableBypassAuth=false` together is the gate.
- **Gating rule:** the public `dai-api` ingress must not be exposed with bypass on or with no working customer identity. Real Entra External ID (tenant, user flow, app registration, audience) must be in place before public exposure. This is a launch blocker, not a provisioning detail (sections 18 and 19 of `cloud-tool-runtime-plan.md`, and `next-platform-architecture-plan.md`).

## 15. build and push sequence

Assumes Azure CLI authenticated, the resource group and ACR created (provisioning slice), and `docker` available. Run from the `dai/` repo root.

1. `az acr login --name crdaiprod`
2. Build and tag the API image with the immutable git sha and the `prod` channel tag:
   `docker build -t crdaiprod.azurecr.io/dai-api:<gitsha> -t crdaiprod.azurecr.io/dai-api:prod platform/dotnet`
3. Build and tag the analyzer image:
   `docker build -t crdaiprod.azurecr.io/dai-analyzer:<gitsha> -t crdaiprod.azurecr.io/dai-analyzer:prod services/agent-service`
4. Push both, sha and channel tags:
   `docker push crdaiprod.azurecr.io/dai-api:<gitsha>` and `:prod`; `docker push crdaiprod.azurecr.io/dai-analyzer:<gitsha>` and `:prod`.
5. Deploy/update the container apps to the new sha tag (analyzer first so the API's dependency is live):
   update `dai-analyzer` to `dai-analyzer:<gitsha>`, confirm internal `/api/ping`, then update `dai-api` to `dai-api:<gitsha>`.
6. Build the Angular client with the production `apiBaseUrl` set to the public `dai-api` URL and deploy to `swa-dai-sports-web` (its own pipeline).

ACR build (`az acr build`) is an acceptable alternative to local `docker build`+`push` and removes the local Docker dependency; either is fine. Use the sha tag on the revision so a rollback is a pin.

## 16. deployment smoke checklist

Mirror the local `compose.smoke.yaml` and `test-sports-dev.ps1` shape, against the cloud edges:

1. `dai-api` liveness: `GET https://<dai-api-fqdn>/health` returns 200 `{"status":"ok"}`.
2. `dai-analyzer` is not publicly reachable: it has no public FQDN; confirm there is no external ingress.
3. Internal reachability: a `dai-api` run that calls the analyzer succeeds (the analyze hop resolves `http://dai-analyzer`). If a direct probe is wanted, exec into `dai-api` and curl `http://dai-analyzer/api/ping` -> 200.
4. Auth gate: an unauthenticated call to a protected `dai-api` endpoint returns 401 (proves `Dev__EnableBypassAuth=false` and real bearer auth are active).
5. SQL connectivity: a run that persists an `AgentRun` succeeds and the row is readable back (`GET /api/agent-runs/{id}/artifact`), proving the Key Vault connection string resolved.
6. End-to-end sports run: one real matchup analyze through `dai-api` -> Tool Gateway -> `dai-analyzer` -> back, producing a `sports_decision_artifact_v2` artifact with a populated `CognitiveProtocol`. This exercises SQL + OpenAI + the gateway together (the chain the local smoke deliberately skipped).
7. Telemetry: confirm `ToolGatewayInvocation` traces appear in `appi-dai-prod` with the run's `X-Agent-Run-Id`.
8. CORS: from the deployed `swa-dai-sports-web` origin, a browser call to `dai-api` succeeds (no CORS error).

## 17. rollback plan

- **Container Apps revisions are the primary rollback.** Each deploy creates a new revision pinned to an immutable image sha tag. To roll back, shift 100% of traffic to the previous known-good revision (or reactivate it) and deactivate the bad one. Because revisions reference the sha tag, not `:latest`/`:prod`, the previous revision keeps pulling the exact prior image. This is a seconds-to-minutes operation with no rebuild.
- **Keep at least the last good revision active or reactivatable** for each app before promoting a new one. Roll the analyzer back first if the analyze hop regresses, then the API.
- **Config rollback:** env var and secret changes are revision-scoped for image-bound config and otherwise re-applied; revert by redeploying the prior revision's template. Key Vault secret values are versioned, so a bad rotated secret can be rolled back to the prior secret version.
- **Static Web Apps** keeps deployment history; roll the client back to the prior build if the production `apiBaseUrl` or bundle is wrong.
- **Database:** no schema change is part of this slice, so rollback is image/config only. If a future slice adds a migration, that slice owns its own forward-fix/rollback plan; do not roll a schema back under a running revision without one.
- **Hard floor:** if a revision rollback does not recover the public edge, scale `dai-api` ingress down / disable public ingress to stop serving a broken edge while diagnosing, rather than leaving a degraded public API up.

## 18. what must be done manually before deploy

These are human actions outside this docs slice and outside Claude. They gate a real deploy.

- **SQL password rotation (gating).** The base `appsettings.json` carried a real SQL password in committed git history (placeholdered by Secrets Hygiene v1, but the historical value is still live until rotated). Rotate the SQL credential, then store the new Azure SQL connection string as the Key Vault secret `connectionstrings-sql`. Do not commit the new value anywhere.
- **Odds API key rotation.** The odds api key was never committed but was surfaced in assistant session transcripts during inspection. Rotate it at the provider as defense-in-depth and store as Key Vault secret `oddsapi-apikey`. Also rotate the OpenAI key if it was similarly exposed and store as `openai-api-key`.
- **Customer auth decision (gating for public exposure).** Decide and stand up Entra External ID: tenant, user flow, app registrations (API + SPA), audience, and confirm the `dai-api` JWT authority shape is correct for External ID (section 14). Public `dai-api` ingress must not go live before this is working with `Dev__EnableBypassAuth=false`.

## 19. what is explicitly deferred

Per `cloud-tool-runtime-plan.md` section 11, called out so they cannot quietly become blockers:

- **MCP.** No MCP adapter in the synchronous run path. The transport seam is documented; the first MCP-bound tool ships only when a node spec calls for it.
- **pgvector.** No Azure Database for PostgreSQL Flexible Server in this slice. pgvector is additive calibration/memory, offline-first, opened to a node only on a calibration finding. SQL Server stays the system of record.
- **Azure Functions.** No timer/event functions (outcome reconciliation re-host, nightly refresh) in this slice. They are post-launch reflex jobs, never on the synchronous run path.
- **AKS / Kubernetes.** Container Apps is the host until proven insufficient. No self-managed Kubernetes.
- **Multi-region.** One region at launch. Cross-region replicas and traffic management are post-launch.
- Also deferred (unchanged from doctrine): API Management, brokers/queues, a dynamic database-backed Tool Registry, per-tenant registry overrides, gateway idempotency-cache and cost-class enforcement (registry fields stay declarative), tenant-tier enforcement, exposing the FastAPI gRPC listener, and splitting the analyze call.

## 20. recommended next implementation slice

**Azure Container Apps provisioning execution (the create slice).** With this plan fixed, the next slice provisions the named resources in `rg-dai-prod` in `eastus2` (or the region the open question resolves to) and deploys, in order:

1. resource group, Log Analytics (`log-dai-prod`), App Insights (`appi-dai-prod`), ACR (`crdaiprod`), Key Vault (`kv-dai-prod`).
2. populate Key Vault with the four secrets (section 10) using the rotated values (section 18).
3. Azure SQL (`sql-dai-prod` / `devcore`) and store its connection string in Key Vault.
4. build and push both images (section 15).
5. create `cae-dai-prod`, deploy `dai-analyzer` (internal ingress, 8000), then `dai-api` (public ingress, 8080) with env vars and Key Vault refs and `AiService__BaseUrl=http://dai-analyzer`.
6. externalize CORS origins and add the web origin; build and deploy `swa-dai-sports-web` with the production `apiBaseUrl`.
7. run the section 16 smoke checklist.

Two small code changes are prerequisites for that slice and can land just ahead of it: externalizing CORS allowed origins to config (section 13), and confirming the JWT authority shape for Entra External ID (section 14). Both are gated by the manual blockers in section 18. Until SQL rotation and the customer auth decision are done, provisioning can proceed up to but not including public exposure of `dai-api`.

## references

- `02 Platform/architecture/cloud-deploy-readiness-v1.md`
- `02 Platform/architecture/cloud-tool-runtime-plan.md`
- `02 Platform/architecture/next-platform-architecture-plan.md`
- `02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
- `06 Execution/handoffs/current-slice.md`
- `dai/compose.smoke.yaml`, `dai/platform/dotnet/Dockerfile`, `dai/services/agent-service/Dockerfile`
