---
title: "Thin Tenant-Scoped Calibration Metrics Endpoint v1"
type: "evidence-report"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Thin Tenant-Scoped Calibration Metrics Endpoint v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - metrics
  - calibration
  - provenance
related:
  - "06 Execution/exports/calibration-outcome-metrics-by-prompt-route-v1.md"
  - "06 Execution/reports/calibration-rows-export-endpoint-v1.md"
---

# Thin Tenant-Scoped Calibration Metrics Endpoint v1

**status:** active doctrine (thin internal GET endpoint composing exporter + metrics calculator; tenant-scoped)
**date:** 2026-06-29

## purpose

Expose the existing prompt-route calibration metrics through a thin, tenant-scoped, internal GET endpoint that
composes the platform-side exporter with the pure metrics calculator. Makes calibration metrics accessible to
operators/internal tooling without building a dashboard, a buyer surface, or any new aggregation logic.

## start state

dai `dfa4032` (main, synced, on origin): `feat(agentruns): platform-side prompt provenance calibration export`
and `feat(agentruns): calibration outcome metrics by prompt route` are pushed; `PromptRouteCalibrationExporter`
(tenant-scoped) and `PromptRouteCalibrationMetricsCalculator` (pure aggregation) exist; `DEFAULT_ALLOWLIST` =
four regimes. dai-vault `b7470fb` (main, synced) with the export + metrics docs. Only the pre-existing untracked
`06 Execution/system-state-synopsis-v1.md` was dirty (unrelated, untouched).

## endpoint route

`GET /api/agent-runs/prompt-route-calibration/metrics` (read-only, no body) on `AgentRunsController`. The path is
a literal segment that does not collide with the existing `{agentRunId:guid}` route (guid-constrained) or the
other agent-run routes.

## tenant-scoping model

`tenantKey` comes from `identity.ResolveAsync(ct)` (the authenticated request/identity context) -- never from a
caller-supplied query/route value. Identity-resolution failure returns 401, matching the neighboring endpoints
(e.g. `GetRecent`). The resolved `ident.TenantKey` is passed to
`PromptRouteCalibrationExporter.ExportAsync(tenantKey, ct)`, which filters AgentRun rows by `TenantKey` BEFORE
returning them. So only the calling tenant's rows are ever loaded; no global/admin or cross-tenant view exists.
Calibration is a tenant-level concern, so scoping is by tenant (not per-user), consistent with the exporter's
design.

## composition flow

```
identity.ResolveAsync(ct) -> ident.TenantKey
  -> PromptRouteCalibrationExporter.ExportAsync(ident.TenantKey, ct)   // tenant-scoped rows from AgentRun + AgentRunOutcome
  -> PromptRouteCalibrationMetricsCalculator.Calculate(rows)            // pure aggregation
  -> PromptRouteCalibrationMetricSummary                               // returned as-is (API-safe)
```

The endpoint is thin and boring: it resolves identity and composes the two existing components. No aggregation
logic lives in the controller; the exporter owns the query + tenant filter; the calculator owns aggregation.

## response schema

Returns `PromptRouteCalibrationMetricSummary` directly (already API-safe -- route metadata + counts only).
Summary: `totalRows`, `reconciledRows`, `unreconciledRows`, `noDecisionRows`, `unknownRouteRows`, `fallbackRows`,
`registryRows`, `liveRows`, `matchedRows`, `unmatchedRows`, `matchRate`, `averageConfidence`, `routes`.
Per route (`routes[]`): `promptRouteKey`, `selectedPromptRecipeId`, `selectedPromptVersion`,
`selectedDataRegime`, `promptSource`, `fallbackReason`, `legacyFallbackUsed`, `totalRows`, `reconciledRows`,
`unreconciledRows`, `noDecisionRows`, `fallbackRows`, `registryRows`, `liveRows`, `matchedRows`, `unmatchedRows`,
`matchRate`, `averageConfidence`, `averageConfidenceMatched`, `averageConfidenceUnmatched`.

No raw model prompts, prompt bytes, artifact prose, full protocol data, or chain-of-thought. The metric records
carry no per-row tenant/user identifiers (the input rows' tenantKey is not surfaced per route).

## empty dataset behavior

A tenant with no sports runs returns a summary with `totalRows = 0` and `routes = []` (HTTP 200), never an error
(proved by a test).

## no-CoT / no-prompt-text guarantee

The metric DTOs have no field for summary / factors / protocol / lean prose. A test seeds OutputJson containing
`SECRET_SUMMARY` / `SECRET_FACTOR` and asserts neither token appears anywhere in the raw endpoint response body.

## buyer-facing impact

None. Internal read-only endpoint; no buyer UX/copy/body, no prompt/template/recipe change, no confidence/model
change. No new metric definitions.

## paid-call status

None.

## tests run

- `dotnet build` (DevCore.Api + tests) -> 0 warnings, 0 errors.
- `dotnet test DevCore.Api.Tests --filter calibration_metrics_endpoint` -> 2 passed (tenant-scoped route metrics
  with a tenant-2 run excluded; empty dataset -> empty summary; no-CoT body assertion).
- `dotnet test DevCore.Api.Tests` (full) -> **971 passed, 0 failed** (was 969; +2).
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- No paid model calls.

## rollback plan

Remove the `GetPromptRouteCalibrationMetrics` endpoint method and the `IPromptRouteCalibrationExporter`
constructor parameter from `AgentRunsController`, and delete the two endpoint tests. Additive change (one
endpoint + one ctor dependency, both already DI-registered); no schema/contract/data change, so removal is clean.

## risks / deferred items

- The provenance column migration `20260629174632_AddAgentRunPromptRouteProvenance` is still unapplied to live
  SQL (dev `devcore-sql`/Docker down across recent slices) -- apply with `dotnet ef database update` (or the
  deploy migration step) before the endpoint runs against live data. In-memory tests build the schema from the
  model and pass without it.
- No pagination/time-window filter on the export yet -- a tenant with very many runs aggregates all of them per
  request; add a date/window filter if volume warrants (deferred).
- No CSV/download form (JSON summary only); a download endpoint is a possible later slice.
- Endpoint is tenant-scoped only; no global/admin calibration view (deliberately out of scope).

## next recommended slice

Apply AgentRun Provenance Migration to Dev SQL v1 (clear the standing migration debt), OR Calibration Metrics
Export Download v1 (CSV/JSON download form of this endpoint), OR Live-Scheduled Starter-Missing Soak v1, OR
Broad Cohort Rerun Grouped by Prompt Recipe v1.

Related: [[calibration-outcome-metrics-by-prompt-route-v1]],
[[platform-side-prompt-provenance-calibration-export-v1]],
[[dotnet-agentrun-prompt-provenance-persistence-v1]], [[default-allowlist-widening-v1]].
