---
title: "Platform-Side Prompt Provenance Calibration Export v1"
type: "export"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Platform-Side Prompt Provenance Calibration Export v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - provenance
  - calibration
  - metrics
related:
  - "06 Execution/exports/prompt-provenance-calibration-export-v1.md"
  - "06 Execution/dotnet-agentrun-prompt-provenance-persistence-v1.md"
---

# Platform-Side Prompt Provenance Calibration Export v1

**status:** active doctrine (calibration export can now read durable AgentRun provenance; JSONL path retained)
**date:** 2026-06-29

## purpose

Shift the prompt provenance calibration export's source of truth toward the durable .NET platform record. The
.NET AgentRun row now persists `PromptRouteProvenanceJson` (.NET AgentRun Prompt Provenance Persistence v1), so
calibration no longer has to depend on transient headers, operator-local JSONL, or analyzer-side scratch files.
This slice adds a platform-side export that reads provenance straight from AgentRun, joined to outcomes, in the
same calibration-ready schema as the python export.

## start state

dai `a9cf539` (main, synced, on origin): both `feat(prompting): add prompt provenance calibration export` and
`feat(agentruns): persist prompt route provenance on the agentrun row` are pushed; AgentRun has nullable
`PromptRouteProvenanceJson`; migration `20260629174632_AddAgentRunPromptRouteProvenance` exists;
`GET /api/agent-runs/{id}/artifact` exposes `AgentRunArtifactDto.PromptRouteProvenance`; `DEFAULT_ALLOWLIST` =
four regimes. dai-vault `c8a99c3` (main, synced) with the prior provenance docs. Only the pre-existing untracked
`06 Execution/system-state-synopsis-v1.md` was dirty (unrelated, untouched).

## source-of-truth shift

Before: calibration read analyzer-side route-provenance JSONL (+ optional operator artifact/outcome dump) via
the python `route_calibration_export`. After: a platform-side .NET export reads provenance from the durable
AgentRun row (the system of record) and joins the durable `AgentRunOutcome`. The python JSONL path is retained
unchanged as a legacy/fallback input (still useful for analyzer-local audit and pre-persistence runs).

## platform-side export source

`DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs`:
- `IPromptRouteCalibrationExporter` / `PromptRouteCalibrationExporter` — internal read-only service over
  `AppDbContext`. `ExportAsync(long tenantKey, CancellationToken)` runs a tenant-scoped LEFT JOIN of `AgentRuns`
  (RunType = sports.matchup.analysis) to `AgentRunOutcomes` (on `AgentRunKey`), `AsNoTracking`, ordered by
  `StartedUtc`, and projects one `PromptRouteCalibrationRow` per run.
- Provenance comes from the durable `AgentRun.PromptRouteProvenanceJson` column, parsed defensively via
  `PromptRouteProvenance.TryParseHeader` (malformed/absent -> null -> unknown route).
- Bounded decision fields come from the run-row `LeanSide` column plus a defensive deserialize of `OutputJson`
  (`Confidence`, `Posture`, `EvidenceRichness`); `AdvertisedStrength` is re-derived via
  `AdvertisedStrengthDeriver`. Model reasoning / prose (summary, protocol, factors) is never read into the row.
- Registered scoped in `Program.cs`. No HTTP endpoint yet (Option B: internal testable service); a future
  endpoint or job can call it.

## JSONL fallback / legacy status

Retained and unchanged. The python `route_calibration_export` + `scripts/export_route_calibration.py` still read
analyzer JSONL (+ optional artifact dump). It remains the path for analyzer-local audit and runs that predate
platform persistence; the platform-side export is now the durable, preferred source for persisted runs.

## output schema

`PromptRouteCalibrationRow` (camelCase JSON keys, matching the python schema):
- identity/run: `agentRunId`, `tenantKey`, `createdUtc`, `competition`, `externalGameId`, `sourceProvider`,
  `homeTeamRef`, `awayTeamRef`, `commenceTime`.
- provenance: `promptSource`, `registryAuthoritativeEnabled`, `legacyFallbackUsed`, `regimeAllowlisted`,
  `routingReason`, `fallbackReason`, `selectedDataRegime`, `selectedPromptRecipeId`, `selectedPromptVersion`,
  `assembledHash`.
- derived: `promptRouteKey` (`recipe@version::regime`, else `unknown`).
- decision: `leanSide`, `confidence`, `advertisedStrength`, `posture`. (evidenceSufficiency / sourceDepthBand are
  available per-run on `/artifact` but deferred for the bulk export to avoid re-deriving source taxonomy here.)
- outcome: `outcomeStatus`, `resultSide` (derived from outcomeStatus: home_win->home, away_win->away,
  draw->draw, else null), `matchedOutcome` (true when an outcome row joined, else null), `reconciledAtUtc`
  (= `AgentRunOutcome.ResolvedUtc`).

No prompt text, prompt bytes, chain-of-thought, or protocol prose is exported.

## grouping keys

Group by `selectedPromptRecipeId`, `selectedPromptVersion`, `selectedDataRegime`, `promptSource`,
`fallbackReason`, `legacyFallbackUsed`, or the combined `promptRouteKey`. Outcome fields enable
route-vs-outcome calibration (e.g. group by `promptRouteKey`, measure `matchedOutcome` / `resultSide`).

## historical / missing / malformed behavior

- Null `PromptRouteProvenanceJson` (historical / header-less / non-MLB runs): route fields null,
  `promptRouteKey` = `unknown`. The run still exports.
- Malformed `PromptRouteProvenanceJson`: `TryParseHeader` swallows the `JsonException` -> null -> unknown route;
  the export never crashes.
- Malformed/blank `OutputJson`: `DeserializeDecision` returns null -> null decision fields; no crash.
- No matching outcome: outcome fields null, `matchedOutcome` null. Nothing fabricated.
- No backfill required.

## tenant-scope behavior

Mandatory. `ExportAsync(tenantKey)` filters `WHERE TenantKey == tenantKey`, so a caller can only export its own
tenant's runs; cross-tenant rows never appear (proved by `is_tenant_scoped`). A future endpoint must pass the
resolved tenant key from the identity context, matching the existing read endpoints.

## migration / local DB status

The provenance column migration `20260629174632_AddAgentRunPromptRouteProvenance` (from the prior slice) is
committed but was **NOT applied to live/dev SQL** in this slice either: the Docker daemon / `devcore-sql`
container was down. This slice adds **no new migration** (it reads the existing column). Verification uses the
EF Core in-memory provider, which builds the schema from the model, so the column is present in tests without
applying the migration. The migration still needs `dotnet ef database update` (or the deploy migration step)
against real SQL on the next platform startup.

## commands

No CLI/endpoint added this slice (internal service). It is invoked in tests via
`new PromptRouteCalibrationExporter(db).ExportAsync(tenantKey, ct)` and is DI-registered for a future
endpoint/job. The python JSONL export commands are unchanged.

## tests run

- `dotnet build` (DevCore.Api + tests) -> 0 warnings, 0 errors.
- `dotnet test DevCore.Api.Tests --filter PromptRouteCalibrationExporterTests` -> 6 passed (durable provenance +
  route key; null -> unknown; malformed -> unknown no-crash; outcome join present/absent; tenant scoping;
  no-chain-of-thought).
- `dotnet test DevCore.Api.Tests` (full) -> **958 passed, 0 failed** (was 952; +6).
- `pytest tests/test_route_calibration_export.py tests/test_route_provenance.py
  tests/test_route_provenance_exposure.py -q` -> 28 passed (JSONL export path intact).
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- No paid model calls.

## buyer-facing impact

None. Internal read-only export service; no buyer UX/copy/body, no endpoint, no prompt/template/recipe change,
no confidence/model change. No chain-of-thought exposed.

## paid-call status

None.

## rollback plan

Delete `DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` and its test, and remove the one DI registration
line in `Program.cs`. Purely additive (new internal service + DI line); no schema/contract/data change, so
removal is clean. The python JSONL export is untouched and remains available.

## risks / deferred items

- The provenance column migration is still unapplied to real SQL (dev container down) -- apply on next deploy.
- `evidenceSufficiency` / `sourceDepthBand` are not in the bulk export (available per-run on `/artifact`);
  deferred to keep the export from re-deriving the source taxonomy.
- No HTTP endpoint/CLI yet -- the export is an internal service; a thin tenant-scoped endpoint or scheduled job
  is the natural follow-up.
- Outcome coverage depends on `AgentRunOutcome` rows existing (reconciliation); un-reconciled runs export null
  outcome (correct).

## next recommended slice

Calibration Outcome Metrics by Prompt Route v1 (aggregate the export into hit-rate / calibration metrics grouped
by `promptRouteKey`), OR a thin tenant-scoped export endpoint/job over `IPromptRouteCalibrationExporter`, OR
Live-Scheduled Starter-Missing Soak v1, OR Broad Cohort Rerun Grouped by Prompt Recipe v1.

Related: [[dotnet-agentrun-prompt-provenance-persistence-v1]], [[prompt-provenance-calibration-export-v1]],
[[prompt-provenance-read-model-exposure-v1]], [[default-allowlist-widening-v1]].
