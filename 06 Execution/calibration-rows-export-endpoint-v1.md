---
title: "Calibration Rows Export Endpoint v1"
type: "evidence-report"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Calibration Rows Export Endpoint v1"
repos:
  dai: "code"
  dai-vault: "docs-only"
tags:
  - calibration
  - export
  - endpoint
  - read-only
related:
  - "06 Execution/exports/calibration-metrics-export-2026-06-30.md"
  - "06 Execution/okf-yaml-front-matter-pattern-v1.md"
---

# Calibration Rows Export Endpoint v1 -- Evidence Report

## purpose

Add a small read-only ROW-level companion to the aggregated metrics endpoint so calibration analysis can inspect
per-run confidence, analyzer-vs-calibrated confidence, evidence richness, advertised strength, regime,
provenance, outcome, and directional-vs-no-decision behavior -- without touching reconciliation logic, prompts,
or buyer surfaces. Smallest change that unlocks calibration rows.

## start state

- `dai`: clean, synced, `d1fbb33` (0/0). `dai-vault`: clean, synced, `aa32def` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4). No paid calls. No reconciliation writes.

## endpoint / helper added

**`GET /api/agent-runs/prompt-route-calibration/rows`** -- the row-level companion to
`.../prompt-route-calibration/metrics`. It reuses the SAME tenant-scoped `PromptRouteCalibrationExporter.
ExportAsync(tenantKey)` the metrics endpoint already calls, but returns the `PromptRouteCalibrationRow[]`
WITHOUT aggregating. Read-only (writes nothing), tenant-scoped via the resolved identity (never a caller-supplied
tenant), not on a buyer surface (admin agent-runs API, same as metrics), empty dataset -> `[]`.

The exporter already strips model reasoning/prose (only route metadata + bounded decision/outcome fields), so no
`summary`/`factors`/chain-of-thought is exposed -- asserted by the endpoint test.

## fields exported (per row)

From the run row + provenance + a defensive parse of the artifact OutputJson:
`agentRunId, tenantKey, createdUtc (StartedUtc), competition (sport), externalGameId (gamePk), sourceProvider,
homeTeamRef, awayTeamRef, commenceTime, promptSource, registryAuthoritativeEnabled, legacyFallbackUsed,
regimeAllowlisted, routingReason, fallbackReason, selectedDataRegime (regime), selectedPromptRecipeId,
selectedPromptVersion, assembledHash, promptRouteKey, leanSide, confidence, advertisedStrength, posture,
outcomeStatus, resultSide, matchedOutcome, reconciledAtUtc`.

**Three fields added this slice** (additive, trailing-optional nullable on `PromptRouteCalibrationRow`; the
metrics calculator ignores them, so metrics output is unchanged):
- `artifactVersion` -- the sports_decision_artifact schema version (v1/v2/v3), for versioned calibration slicing.
- `evidenceRichness` -- the grounded-signal-category count (the **evidence-sufficiency proxy**; see below).
- `analyzerConfidence` -- the analyzer's pre-calibration confidence, to compare against the calibrated
  `confidence` (overconfidence signal).

**directional vs no-decision** is derivable per row: `leanSide == null` -> no-decision; else directional.

## fields unavailable and why

- **evidenceSufficiency (as a distinct string/enum):** there is no separate sufficiency field on the persisted
  artifact contract. The closest grounded signal is `evidenceRichness` (an integer count of grounded signal
  categories from the evaluator step), which this export now exposes as the sufficiency proxy. A dedicated
  sufficiency enum would need an artifact-contract change -- out of scope, not invented.
- **assembledHash / recipe / version / artifactVersion are null for some rows** -- honest, not a bug: legacy
  pre-provenance runs and older artifact versions do not carry them (nested/versioned OutputJson). Nulls are
  returned as-is; nothing is inferred (asserted by `new_calibration_fields_are_null_when_absent_from_artifact`).

## code changes

- `platform/dotnet/DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` -- 3 trailing nullable fields on
  `PromptRouteCalibrationRow` (with `= null` defaults; all construction is via named args, so no existing
  call-site breaks) + populate them in `Shape(...)` from the parsed `AgentRunExecutionResult`
  (`ArtifactVersion`, `EvidenceRichness`, `AnalyzerConfidence`).
- `platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs` -- new
  `GetPromptRouteCalibrationRows` action (`GET .../prompt-route-calibration/rows`), mirroring the metrics
  endpoint's identity resolution + tenant scoping; returns the exporter rows directly.

No change to the metrics calculator, provenance contract, DB schema, prompts, allowlist, or any buyer surface.

## tests / checks run

- NEW exporter unit tests (in-memory EF): `exports_artifact_version_evidence_richness_and_analyzer_confidence`
  (rich artifact -> fields populated, calibrated confidence unchanged);
  `new_calibration_fields_are_null_when_absent_from_artifact` (absent -> null).
- NEW integration tests (WebApplicationFactory + HttpClient):
  `calibration_rows_endpoint_returns_tenant_scoped_rows_without_prose` (tenant 1's 2 rows only, tenant 2 absent,
  no SECRET_SUMMARY/SECRET_FACTOR leak, confidence + regime + promptSource present, directional + no-decision
  both present, reconciled row carries its matched outcome); `calibration_rows_endpoint_empty_dataset_returns_
  empty_list`.
- `dotnet build DevCore.Api` -> 0 warn / 0 err.
- `dotnet test --filter "PromptRouteCalibration|PromptRouteProvenance|PromptRouteKeyFallback|AgentRunsController
  Tests"` -> **103 passed** (99 prior + 4 new). The existing metrics endpoint tests still pass unchanged.

## sample row count + aggregate cross-check (dev data, tenant 1)

Live cross-check that the rows reproduce/explain the aggregate:

| check | rows endpoint | metrics endpoint | match |
|---|---|---|---|
| total rows | 263 | totalRows 263 | ✔ |
| promptSource=registry | 27 | registryRows 27 | ✔ |
| promptSource=live | 1 | liveRows 1 | ✔ |
| rows with an outcome | 84 | reconciledRows 76 + noDecisionRows 8 = 84 | ✔ |

New fields populated where present (nulls where the artifact lacks them): artifactVersion non-null 185/263,
evidenceRichness 203, analyzerConfidence 205, confidence 263. Row-level directional split (not previously
exposed): **directional 171 / no-decision 92**.

## paid-call status

**None.** Build/tests + read-only endpoint calls on dev data. No model call, no DB writes.

## buyer-facing impact

**None.** The endpoint is on the admin agent-runs API (same as the metrics endpoint), tenant-scoped, read-only,
and exposes no prose/chain-of-thought (exporter-stripped, test-asserted). `SportsAnalysisResponse` and all buyer
routes/copy are untouched.

## confirmation -- untouched surfaces

Prompts / prompt registry: **untouched.** DEFAULT_ALLOWLIST: **unchanged** (4). Buyer copy/routes: **untouched.**
DB schema: **untouched** (no migration -- fields read from existing OutputJson). Reconciliation state:
**untouched** (endpoint writes nothing; outcomes still 84, backlog still 0 reconciled).

## risks / deferred items

- No paging/limit on `/rows` -- fine at 263 rows; add a `take`/cursor if the dataset grows large (deferred).
- `evidenceRichness` is a proxy for sufficiency, not a first-class sufficiency enum; a dedicated field would need
  an artifact-contract change (deferred, out of scope).
- Rows still reflect the pending backlog (20 unreconciled) -- directional match reads broaden only when those
  games settle.

## recommended next slice

**Outcome Reconciliation Follow-up v4 + Calibration Delta v1** once a StatsAPI probe shows >=1 backlog game
`Final` (the 06-30 slate was In Progress at last probe -- Finals imminent): reconcile the settled subset and use
this new `/rows` endpoint to compute a before/after calibration delta (confidence vs outcome, analyzerConfidence
vs calibrated, per-regime accuracy) -- the first deep calibration read the row export now makes possible.
