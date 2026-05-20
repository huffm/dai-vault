# cloud and tool runtime plan

**date:** 2026-05-20
**status:** doctrine. forward plan for cloud target and tool governance. no runtime code changes in this slice.
**scope:** how DAI hosts its services, governs tool calls, and stages future infrastructure without taking launch risk.

## what this document is

The Cognitive Protocol Runtime defines the cognitive shape of every run: a shared decision artifact moves through Perceive, Interrogate, Discern, Decide, and Synthesize, and each node has a fixed list of allowed tools, scripts, model calls, and artifact fields it may touch (see `cognitive-factory/cognitive-protocol-runtime.md` and `cognitive-factory/protocol-node-specs.md`). The runtime tells you what cognition is allowed. This document tells you what infrastructure cognition runs on, and what governs the tools cognition is allowed to invoke.

It is launch-conscious. It names a small launch-friendly cloud target, locks the Tool Gateway as the single authority over tool calls, and lists the items that are explicitly deferred until after launch so they cannot quietly grow into blockers.

## the runtime in one sentence

Deterministic .NET code in `DevCore.Api` orchestrates a sports run, calls typed HTTP clients for retrieval, calls FastAPI for the analyze model call, scores deterministically with `SportsEvaluator`, and composes the artifact with `SportsComposer`. Persistence is SQL Server today (`AgentRun.OutputJson`). The Angular sports app is a static client.

## 1. launch-friendly cloud target

Launch ships on Azure with a small, boring set of services. The target is to remove "where does this run" as a blocker, not to adopt every Azure product available.

The launch surface is:

- Azure Container Apps for the .NET orchestrator (`DevCore.Api`) and the FastAPI analyzer (`services/agent-service/`). One container per service. Internal HTTP between them on a private revision-to-revision route. Public ingress only on the .NET API.
- Azure SQL (or a managed SQL Server) as the system of record for `AgentRun`, `Sports`, `Teams`, `Users`, and `UserIdentity`. No schema change required vs current dev SQL.
- Azure Static Web Apps (or Azure Storage static site behind a CDN) for the Angular sports app. The Angular client never holds secrets.
- Azure Key Vault for provider keys (odds api, espn, future MCP-bound tools, FastAPI shared secret).
- Azure Application Insights for telemetry, with correlation on `X-Agent-Run-Id` (canonical) and `X-Correlation-Id`.
- Azure Functions for bounded scheduled and event-driven work (see section 6). Not on the synchronous run path.
- Entra External ID as the customer identity provider, as already planned in `next-platform-architecture-plan.md`.

What is intentionally not in the launch surface:

- AKS or any self-managed Kubernetes (see section 10).
- A separate vector database product. pgvector inside a managed Postgres instance is the memory layer (see section 8) and is additive to SQL, not a replacement.
- Service Bus, Event Grid, Kafka, or any broker. The synchronous .NET to FastAPI call is HTTP and stays HTTP for launch.
- API Management. The .NET API ingress is the public edge.
- Multi-region. One region at launch. Cross-region read replicas and traffic manager are post-launch.

This is the smallest cloud footprint that lets DAI ship a paying product, scale to early tenants, and keep options open.

## 2. what stays in .NET

The .NET orchestrator (`DevCore.Api`) keeps every responsibility it has today and gains the Tool Gateway. Specifically:

- pipeline coordination: `AgentRunService.ExecuteAsync` and `ExecuteSportsMatchupAsync` continue to dispatch retrieve, analyze, evaluate, compose in order.
- typed retrieval clients: `OddsScheduleClient`, `OddsMarketClient`, `EspnBasketballScheduleClient`, `MlbStarterClient` and any future provider client.
- deterministic evaluation: `SportsEvaluator` owns calibrated `Confidence` and `ConfidenceBand`. The model never owns the number.
- deterministic synthesis: `SportsComposer` and `CognitiveProtocolBuilder` own Synthesize.Integrate, Synthesize.Compose, Synthesize.Deliver and the deterministic Probe (`BuildProbe`).
- quality checks: `SportsQualityChecker` and the calibration flags.
- identity, tenancy, billing surface: `IdentityResolver`, `DevProvisionController`, Stripe webhook endpoints, tier enforcement when it lands.
- persistence: `AgentRun`, `OutputJson`, reference data tables.
- the Tool Gateway and the Tool Registry (sections 4 and 5). These are .NET because the .NET orchestrator is the only place that already sees the full run, the tenant, the protocol node it is serving, and the artifact field it is about to write.

The rule of thumb: anything that touches the artifact, decides what tool is allowed, or stamps deterministic numbers stays in .NET.

## 3. what stays in FastAPI

The FastAPI analyzer service (`services/agent-service/`) keeps a narrow job: take the validated input plus retrieved context blocks, run the bounded model call, and return the analyze response.

Specifically:

- `app/routes/sports.py` validates the competition code and dispatches by family.
- `app/services/sports_analyzer.py` holds the prompt body and the bounded model call. This is the only model call on the synchronous run path today.
- the FastAPI service does not call the odds api, does not call espn, does not write to SQL, and does not own the artifact shape. It returns `SportsAnalysisResponse` and stops.
- FastAPI is a hot path. It is the right place for the model call and any future prompt-side splits of the 12 cognitive micro-actions, but it is not the place for new tool surfaces, new providers, or new persistence.

The rule of thumb: FastAPI runs prompts. It does not run tools.

## 4. tool gateway responsibilities

The DAI Tool Gateway is the authority. Every tool call, retriever call, FastAPI call, future MCP call, and future Azure Function invocation goes through it. No protocol node, no model call, and no future agent gets to widen its own permissions.

The Tool Gateway lives in .NET, inside `DevCore.Api`, alongside `AgentRunService` and the typed clients. It is a thin layer, not a framework. Its responsibilities:

- enforce the per-node allowed-tools list from `protocol-node-specs.md`. A call from Perceive.Detect cannot invoke a tool that the spec assigns only to Interrogate.Probe.
- resolve a tool id to a concrete handler: a typed HttpClient, an internal service call, a FastAPI route, an Azure Function trigger, a future MCP adapter.
- inject the canonical correlation headers on every outbound call: `X-Agent-Run-Id` (canonical), `X-Correlation-Id`, tenant key.
- enforce idempotency where the tool registry declares it. Repeat calls within a run id and tool id return the cached result, not a duplicate provider hit.
- enforce cost class: a tool flagged "expensive" or "external paid" can be rate-limited or blocked by tenant tier without changing the protocol node code.
- emit telemetry per tool call: tool id, protocol node, run id, tenant key, duration, outcome, cache hit, cost class. This is the surface the calibration loop reads from.
- read secrets from Key Vault. No secret ever lives on the artifact, in `OutputJson`, or in a log line.
- fail closed. An unknown tool id, an unauthorized node, or a tool whose registry entry is missing returns an error to the orchestrator. The artifact still composes through the `ComposeFailedRun` path.

What the gateway is not:

- not an LLM agent.
- not a decision layer.
- not a workflow engine.
- not a place to put niche-specific business rules. The gateway carries permission and transport. Niche rules live in retrievers, prompts, posture vocabularies, and scoring thresholds.

The first concrete implementation does not need to rewrite the typed HttpClients. It wraps them. `OddsMarketClient.GetFootballSpreadAsync` becomes `IToolGateway.InvokeAsync(toolId: "market.football.spread", input, runContext)`, which dispatches to the existing typed client. The wrapping is the slice; the providers are not rewritten.

## 5. tool registry v1 shape

The Tool Registry is the manifest the Tool Gateway reads from. v1 is small and static. It lives in code, not in a database, and it does not require any new table at launch.

Each tool entry carries:

- `tool_id`: stable string, e.g. `market.football.spread`, `market.basketball.spread`, `schedule.basketball.rest`, `starters.mlb`, `analyze.sports`, `eval.signal_quality`, `eval.signal_followup`.
- `kind`: one of `retriever`, `analyzer`, `evaluator`, `composer-helper`, `function`, `mcp-adapter` (future).
- `transport`: one of `inproc-net`, `http-internal`, `http-external`, `azure-function`, `mcp` (future).
- `handler`: the concrete .NET binding (interface plus implementation) or remote target.
- `allowed_protocol_nodes`: the closed list of cognitive nodes that may invoke this tool. Read from `protocol-node-specs.md` and mirrored here. Examples: `analyze.sports` is allowed for the analyze model call only; `market.football.spread` is allowed for the retrieve stage only and is consumed by Perceive.Detect / Perceive.Frame / Perceive.Aim through the retrieval output (those nodes do not call the tool themselves; the spec records the consumption path).
- `secrets_scope`: which Key Vault secrets the handler may read.
- `idempotency`: one of `per_run`, `per_run_input`, `none`. `per_run_input` is the default for retrieval; `per_run` is correct for the analyze call.
- `cache_ttl`: seconds. `0` means do not cache.
- `cost_class`: `free`, `cheap`, `paid_external`, `model_call`. Used for tenant tier enforcement and telemetry.
- `tenant_tier_minimum`: optional. Lowest tier that may invoke this tool. Null at launch for every existing tool.
- `calibration_hooks`: list of metric ids the gateway emits when this tool runs. Pre-seeded with what the calibration reports already track.

The registry shape is intentionally a static manifest in v1 because the only tools that exist are the ones already in code. The shape is designed so that a database-backed dynamic registry, a tenant-scoped allowlist, and a per-tier policy can be added later without rewriting the gateway interface.

## 6. azure functions role

Azure Functions are for bounded reflex jobs and scheduled or event-driven tools. They are not on the synchronous run path.

Good fits for launch and early post-launch:

- the outcome reconciliation harness (`reconcile-calibration-outcomes.ps1`) shape, re-hosted on a timer trigger so reconciliation runs daily after game completion without a person running a script.
- nightly reference data refresh: pulling new team rosters, schedule updates, and competition catalog entries.
- low-priority calibration export rollups.
- Stripe webhook ingestion if it gets noisy enough to warrant a dedicated function rather than the .NET controller.
- retry jobs for transient external provider failures during a run, where a re-attempt is honest and bounded.

Functions invoked through the Tool Gateway behave like any other tool: a `tool_id`, an `allowed_protocol_nodes` list (usually empty for purely platform jobs), and full telemetry. A function that triggers on its own timer is platform plumbing and does not need a protocol-node binding; a function that a node may invoke does.

What does not belong in Functions:

- the analyze model call (it stays in FastAPI on Container Apps so cold start is not a latency tax on every run).
- retrieval that a cognitive node depends on synchronously (cold start tax again, and a function failure mid-run is a worse failure mode than a typed HttpClient timeout).
- any path that needs the full run context to make a permission decision.

## 7. azure container apps role

Container Apps is the launch host for the .NET API and the FastAPI analyzer. Reasons:

- both are long-running HTTP services; this is what Container Apps is for.
- scale-to-zero is allowed for the FastAPI service in low-traffic windows, with the understanding that the .NET API stays warm so the first call of the day pays cold start only on the analyzer.
- revision-based deploys give safe rollouts without an AKS investment.
- the networking story (private internal HTTP, public ingress on the .NET API only) is built in.
- no Kubernetes operator knowledge required.

The contract between the two services is unchanged from local dev: POST `http://{fastapi-internal}/api/sports/analyze` with `X-Correlation-Id` and `X-Agent-Run-Id`. The internal hostname becomes the Container App internal DNS instead of `127.0.0.1:8001`. No code change in the wire contract.

If a future hot retrieval path benefits from horizontal scale beyond what Container Apps gives us, it is hosted as its own Container App or moved behind the Tool Gateway as a separate Function. AKS is not the answer to that problem at launch (section 10).

## 8. postgres and pgvector memory role

Postgres with pgvector is the memory and search layer. It is additive to SQL Server, not a replacement for the primary transactional store before launch.

What lives in pgvector:

- semantic memory of prior runs, indexed by competition, tenant, and node. Useful for future "have we seen this matchup shape before" lookups and for analyst-side review tooling.
- calibration corpora: embeddings of analyze responses, signal coverage shapes, and outcome reconciliations. Used by the calibration loop to find similar runs, not by the synchronous run path.
- niche knowledge packs once they exist: per-niche document corpora that a future Probe-side tool may consult.

What does not live in pgvector at launch:

- `AgentRun`, `OutputJson`, `Sports`, `Teams`, `Users`, `UserIdentity`. Those stay in SQL Server because nothing in the run path needs vector search and rewriting the primary store is launch risk for no launch benefit.
- billing rows. Stripe is the truth (`CLAUDE.md`), with a thin reflection in SQL Server.

How the synchronous run path reaches pgvector when it eventually does:

- through the Tool Gateway, as a tool of kind `retriever` and `cost_class = cheap`.
- `allowed_protocol_nodes` is initially empty for synchronous use and is opened only when a calibration finding supports the addition. The first realistic opening is for Interrogate.Probe, where prior-run lookups can inform investigation candidates.

Pgvector is hosted as Azure Database for PostgreSQL Flexible Server. Same region as Container Apps. No separate vector product is purchased.

## 9. mcp future adapter role

MCP is a future adapter, not the core runtime. The Tool Gateway is the core runtime. MCP enters DAI as one of the `transport` values in the tool registry, alongside `inproc-net`, `http-internal`, `http-external`, and `azure-function`.

What this means in practice:

- a future MCP-bound tool registers in the Tool Registry with `transport = mcp`, an `allowed_protocol_nodes` list, a `cost_class`, and an idempotency rule, just like any other tool.
- the Tool Gateway calls the MCP server through a typed MCP client living inside .NET. The protocol node does not learn that the tool happens to be MCP-bound. The transport is hidden.
- MCP servers DAI publishes (for example, an analyst-side server that exposes calibration queries) are governed by the same registry and the same `allowed_protocol_nodes` rules. There is no separate permission system.
- DAI does not become an MCP-first product. MCP is a way to plug in well-known external capabilities (third-party data, analyst tools, future partner integrations) without writing a bespoke typed client for each one.

What MCP is not:

- not the orchestrator.
- not a replacement for typed HttpClients in .NET. Those stay.
- not a path for a model to choose its own tools. The gateway, not the model, decides what a node may invoke.

## 10. aks and kubernetes future role

AKS is future infrastructure, not required for launch. It enters the picture only if and when one of the following is true:

- DAI is running multi-region with co-located stateful workloads that Container Apps cannot host cleanly.
- a single Container App revision cannot absorb a hot workload (sustained, not bursty) and we need fine-grained node-level control to keep cost honest.
- a partner integration requires hosting infrastructure we control end to end (network policies, sidecars, ingress controllers) that Container Apps cannot express.

None of those is true at launch. The migration story is straightforward when it is needed: both services are already containerized, both already use environment-based configuration, and both already authenticate the same way they would inside AKS. The .NET API and FastAPI services move to AKS with no contract change; the Tool Gateway and Tool Registry travel with the .NET service.

The cost of staying on Container Apps too long is small. The cost of standing up AKS too early is a platform team DAI does not have at launch.

## 11. what is explicitly deferred until after launch

Calling these out so they cannot quietly become blockers:

- AKS or any self-managed Kubernetes. Container Apps until proven insufficient.
- a real MCP integration in the synchronous run path. The adapter shape is documented; the first MCP-bound tool ships only when a node spec calls for it.
- replacing SQL Server with Postgres for primary storage. Postgres is pgvector memory only, in the additive role above.
- multi-region. One region at launch.
- a queue or broker (Service Bus, Event Grid, Kafka). The run path is synchronous HTTP.
- API Management. The .NET API ingress is the edge.
- a dynamic, database-backed Tool Registry. v1 is a static manifest in code.
- per-tenant Tool Registry overrides. Tiers can be added later; launch ships with one effective policy.
- splitting the 12 cognitive micro-actions into separate model calls or separate FastAPI routes. One analyze call today, one analyze call at launch. Splitting is a future calibration decision, not a cloud decision.
- moving Perceive.Detect or Perceive.Frame to deterministic retriever-side extraction. The node specs allow it. It is not on the launch path.
- niche-specific posture vocabularies. The current closed enum (`play`, `pass`, `monitor`, `wait`, `compare`, `avoid`) stays at launch.
- adding NCAAW or college baseball as supported competitions. They remain out of scope as noted in `protocol-node-specs.md`.
- outcome reconciliation as a fully automated learning loop. The harness exists and the function-hosted version is post-launch; calibration evidence still drives confidence rules manually until the loop is proven.
- payment-tier enforcement on tool calls. Registry has the field. Enforcement turns on after Stripe webhook truth is wired through.

## 12. recommended next slices

Concrete, sequenced, and shippable. Each slice is small enough to land cleanly and lays the groundwork for the next one.

1. **tool gateway skeleton in .NET.** Introduce `IToolGateway`, `ToolRegistry` (static manifest), `ToolInvocationContext` (run id, tenant key, protocol node, tool id, correlation), and a thin pass-through implementation. Wrap exactly one existing tool first: `OddsScheduleClient.GetEventsAsync` becomes `IToolGateway.InvokeAsync("schedule.matchup_dates", ...)`. Behavior unchanged. Tests cover registry lookup, allowed-node enforcement, and correlation header injection. No FastAPI change. No database change.

2. **register the remaining typed clients behind the gateway.** Wrap `OddsMarketClient` (football and basketball spread), `EspnBasketballScheduleClient`, `MlbStarterClient`, and the FastAPI analyze call. Each wrap is a separate small change with tests. After this slice every external call goes through the gateway and emits gateway telemetry.

3. **container apps deploy slice.** Package `DevCore.Api` and `services/agent-service/` as containers, provision the Container Apps environment, wire internal DNS, move secrets to Key Vault, point Application Insights at both services. The .NET to FastAPI hop uses internal Container Apps DNS instead of `127.0.0.1:8001`. Smoke test parity with `test-sports-dev.ps1`.

4. **pgvector memory landing.** Provision Azure Database for PostgreSQL Flexible Server with pgvector, build the calibration-side ingestion (offline first, no synchronous run dependency), and add a read-only review surface for prior runs. No protocol node opens against it yet.

5. **first azure function tool: outcome reconciliation timer.** Re-host `reconcile-calibration-outcomes.ps1` as a timer-triggered function that runs the day after a calibration batch. It calls back into the .NET API through the existing endpoints. Tool registry gains its first `transport = azure-function` entry, even though no protocol node invokes it directly.

6. **probe enrichment via gateway, behind a feature flag.** Open `Interrogate.Probe` to a single gateway-bound lookup against pgvector calibration memory, behind a per-tenant flag. The deterministic template path remains the default. This is the first time a cognitive node consumes a gateway-bound tool that is not in the original typed-client set, and it is the first realistic test of the "open `allowed_protocol_nodes` deliberately" rule.

After slice 6, MCP adapters become a reasonable next step because the transport seam, the registry, and the per-node permissioning are all proven by real code, not by doctrine. Until then, MCP stays on paper.
