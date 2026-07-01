---
title: ".NET AgentRun Prompt Provenance Persistence v1"
type: "evidence-report"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: ".NET AgentRun Prompt Provenance Persistence v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - provenance
  - metrics
  - observability
related:
  - "06 Execution/reports/prompt-provenance-read-model-exposure-v1.md"
  - "06 Execution/reports/phase-3-2-global-prompt-routing-hardening-v1.md"
---

# .NET AgentRun Prompt Provenance Persistence v1

**status:** active doctrine (route provenance captured from the analyzer header, persisted on the AgentRun row, exposed read-only)
**date:** 2026-06-29

## purpose

Turn analyzer-side prompt route provenance from a transient header / analyzer-side JSONL audit aid into durable
platform metadata on the .NET system of record. The FastAPI analyzer already emits the route provenance on the
`X-Prompt-Route-Provenance` response header (Prompt Provenance Read-Model Exposure v1). This slice has the .NET
platform capture that header, persist it on the AgentRun row, and expose it through the internal artifact read
model, so calibration can group durable platform data by prompt recipe / version / regime / source / fallback
without depending on transient headers or analyzer JSONL.

## start state

dai `34541d4` (main, synced): Prompt Provenance Read-Model Exposure v1 and Calibration Export v1 pushed; FastAPI
exposes the `X-Prompt-Route-Provenance` header; `DEFAULT_ALLOWLIST` exactly four regimes. dai-vault `8f14c13`
(main, synced) with the prior routing/provenance docs. Only the pre-existing untracked
`06 Execution/system-state-synopsis-v1.md` was dirty (unrelated, untouched).

## why platform-side persistence is needed

The analyzer is the producer of route provenance; the .NET platform is the durable system of record for agent
runs (the AgentRun row + OutputJson artifact). Until now provenance lived only on a transient response header
and an optional analyzer-side JSONL sink, so durable calibration depended on out-of-band data. Persisting it on
the AgentRun row makes provenance a first-class, queryable platform fact, consistent with how game identity and
lean side are already denormalized for calibration.

## fastapi header contract

Header: `X-Prompt-Route-Provenance`. Value: the JSON projection of the python `PromptRouteProvenance` (camelCase
keys): `promptSource`, `registryAuthoritativeEnabled`, `legacyFallbackUsed`, `regimeAllowlisted`,
`routingReason`, `fallbackReason`, `selectedDataRegime`, `selectedPromptRecipeId`, `selectedPromptVersion`,
`assembledHash`, `agentRunId`, `competition`, `createdUtc`. The header is OPTIONAL: MLB-only and emitted only by
the analyzer; a missing header is normal for non-MLB runs and older analyzers.

## .NET capture location

`DevCore.AiClient/FastApiClient.cs`: `AnalyzeSportsMatchupWithProvenanceAsync` performs the same POST as before
and additionally reads the response header into a typed `DevCore.AiClient/PromptRouteProvenance` via
`PromptRouteProvenance.TryParseHeader` (defensive: missing/blank/malformed -> null, never throws). The tool
output type became `SportsAnalysisResult(SportsAnalysisResponse Response, PromptRouteProvenance? RouteProvenance)`
-- provenance is carried BESIDE the response, never inside `SportsAnalysisResponse`. `AnalyzeSportsMatchupAsync`
remains as a thin back-compat wrapper returning just the response.

Threading: `AnalysisSportsMatchupReadHandler` (tool output `SportsAnalysisResult`) -> `SportsAnalyzer` records
`analysis.Response` and `analysis.RouteProvenance` on the artifact -> `AgentRunService` carries it on
`AgentRunExecution.RouteProvenance` -> the controller persists it. This mirrors the existing run-row-adjacent
metadata pattern used for `GameIdentity` and `MarketSnapshot` (never serialized into OutputJson).

## persistence location

New nullable column `AgentRun.PromptRouteProvenanceJson` (`string?`, `nvarchar(max)`), holding the JSON
projection. Migration `20260629174632_AddAgentRunPromptRouteProvenance` (AddColumn + Down DropColumn, nullable,
no backfill) -- mirrors the existing single-column-addition migrations. Written in
`AgentRunsController.Create` on the success path beside `ApplyGameIdentity`, via `SerializeRouteProvenance`
(null in -> null column). Stored as JSON, not in `OutputJson`, so it never touches the artifact body and can be
queried/grouped without parsing the artifact.

## read-model / API exposure

`GET /api/agent-runs/{id}/artifact` -> `AgentRunArtifactDto.PromptRouteProvenance` (a new trailing optional
`PromptRouteProvenance?`, defaulting null). The controller reads the run-row column into the projection and
`DeserializeRouteProvenance` (defensive: null/malformed -> null) maps it onto the DTO. This is the internal
dev/diagnostic inspection surface (same one that exposes game identity, source depth, etc.) -- NOT a buyer
surface. Tenant scoping is unchanged (the query still filters `TenantKey` + `RequestedByUserKey`).

## schema / metadata shape

The persisted/exposed shape is the `PromptRouteProvenance` record: the ten route fields plus the run anchor
(`agentRunId`, `competition`, `createdUtc`). Metadata about prompt SELECTION only -- never prompt text, model
prompt bytes, or chain-of-thought.

## missing / malformed header behavior

- Missing header: `RouteProvenance` is null; the column is null; the run completes normally.
- Malformed header: `PromptRouteProvenance.TryParseHeader` swallows the `JsonException` and returns null; the
  analysis is never failed by a bad header. The controller's `DeserializeRouteProvenance` is equally defensive
  on read.
- The header contract is intentionally NOT expanded here; the current projection is small. Header size is a
  deferred concern (documented), not a risk at this size.

## historical / null behavior

Runs created before this slice have a null `PromptRouteProvenanceJson` column; `/artifact` returns
`PromptRouteProvenance = null` for them and never throws. No backfill (the column is nullable; absence reads
cleanly). Non-MLB runs also carry null (the analyzer emits the header for MLB only).

## tenant-scope behavior

Unchanged. Persistence happens within the existing tenant/user-scoped create flow; the `/artifact` read keeps the
`AgentRunId == id && TenantKey == tenant && RequestedByUserKey == user` filter. Provenance adds no cross-tenant
visibility.

## tests run

- `dotnet test DevCore.Api.Tests` (full suite) -> **952 passed, 0 failed**.
- Targeted: `--filter "SportsAnalyzerTests|ToolGatewayAnalyzeTests|prompt_route_provenance|PromptRouteProvenance"`
  -> all passed (capture-from-header, no-header-null, persist + read-model surface, missing -> null, malformed
  parse safety).
- `dotnet build` (DevCore.Api + tests) -> 0 warnings, 0 errors.
- Python boundary untouched this slice (the header was shipped earlier); not re-run.

New/updated tests: `PromptRouteProvenanceTests` (defensive parse: valid / live-fallback / missing+malformed ->
null without throwing); `SportsAnalyzerTests` (header captured onto the artifact; no header -> null);
`ToolGatewayAnalyzeTests` (updated to the `SportsAnalysisResult` tool output; provenance null when no header);
`AgentRunsControllerTests` (create persists provenance to the run-row column and `/artifact` surfaces it; absent
provenance -> null on the read model; never written into OutputJson).

## buyer-facing impact

None. `SportsAnalysisResponse` (the buyer/analyzer response body) is unchanged; provenance lives on a run-row
column + the internal artifact inspection DTO. No buyer UX/copy/pricing/billing/auth/tenant/dashboard/confidence
/model change; no chain-of-thought exposed.

## paid-call status

None. Provenance is pure metadata captured from an existing response header; no model call is added.

## rollback plan

Revert the .NET diff and drop the migration: `dotnet ef migrations remove` (or apply the `Down()` DropColumn) for
`AddAgentRunPromptRouteProvenance`; revert the tool output type back to `SportsAnalysisResponse` (handler,
registration, SportsAnalyzer); remove `PromptRouteProvenanceJson` from the entity + the DTO/controller wiring;
delete `DevCore.AiClient/PromptRouteProvenance.cs`. The column is additive and nullable, so even leaving it in
place is harmless if only the read/write wiring is reverted.

## risks / deferred items

- DEFAULT allowlist / prompt routing unchanged; the platform only persists + exposes provenance, never selects
  prompts (the architectural boundary held).
- Header size growth is a deferred concern (current projection is small; the contract was not expanded).
- The migration is generated but NOT applied to any live SQL Server in this slice (the dev SQL container was
  down); apply it on the next platform deploy. In-memory tests build the schema from the model and pass without
  it.
- Calibration export (Prompt Provenance Calibration Export v1) still reads analyzer-side JSONL; a follow-up can
  switch it to read platform-persisted provenance now that it is durable.

## next recommended slice

Platform-Side Prompt Provenance Calibration Export v1 (re-point the export at the durable AgentRun column instead
of analyzer JSONL), OR Live-Scheduled Starter-Missing Soak v1, OR Broad Cohort Rerun Grouped by Prompt Recipe v1.

Related: [[prompt-provenance-read-model-exposure-v1]], [[prompt-provenance-calibration-export-v1]],
[[default-allowlist-widening-v1]], [[phase-3-2-global-prompt-routing-hardening-v1]].
