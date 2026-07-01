---
title: "Calibration Outcome Metrics by Prompt Route v1"
type: "export"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Calibration Outcome Metrics by Prompt Route v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - calibration
  - metrics
  - provenance
  - outcome
related:
  - "06 Execution/exports/platform-side-prompt-provenance-calibration-export-v1.md"
  - "06 Execution/exports/prompt-provenance-calibration-export-v1.md"
---

# Calibration Outcome Metrics by Prompt Route v1

**status:** active doctrine (pure metrics calculator over platform-side calibration rows; no endpoint yet)
**date:** 2026-06-29

## purpose

Add the first calibration metrics layer: aggregate platform-side prompt provenance calibration rows
(`PromptRouteCalibrationRow`) into compact route-level outcome metrics grouped by prompt route. This turns the
export rows from Platform-Side Prompt Provenance Calibration Export v1 into succinct, inspectable performance
visibility per prompt recipe / version / regime / source / fallback. Metrics aggregation only -- not a
dashboard, endpoint, or buyer surface.

## start state

dai `bda6c4d` (main, synced, on origin): `feat(agentruns): persist prompt route provenance on the agentrun row`
and `feat(agentruns): platform-side prompt provenance calibration export` are pushed;
`PromptRouteCalibrationExporter` reads durable AgentRun provenance + LEFT JOIN AgentRunOutcome, tenant-scoped;
`DEFAULT_ALLOWLIST` = four regimes. dai-vault `53bfd5e` (main, synced) with the prior provenance/export docs.
Only the pre-existing untracked `06 Execution/system-state-synopsis-v1.md` was dirty (unrelated, untouched).

## metric component names

`DevCore.Api/AgentRuns/PromptRouteCalibrationMetrics.cs`:
- `PromptRouteCalibrationMetricsCalculator` -- static, PURE: `Calculate(IEnumerable<PromptRouteCalibrationRow>)
  -> PromptRouteCalibrationMetricSummary`. No DB, no i/o, no model call.
- `PromptRouteCalibrationMetricRow` -- per-route metrics.
- `PromptRouteCalibrationMetricSummary` -- grand totals + the route rows.

Component boundary (kept separate per the architecture intent): the exporter owns row production + tenant
scoping; this calculator only aggregates rows. A future endpoint/job composes them:
`PromptRouteCalibrationExporter.ExportAsync(tenantKey) -> PromptRouteCalibrationMetricsCalculator.Calculate(rows)`.

## input row source

`PromptRouteCalibrationRow` records from `PromptRouteCalibrationExporter` (durable AgentRun provenance + joined
AgentRunOutcome). The calculator consumes already-shaped rows; it never re-parses provenance JSON or queries the
DB.

## grouping keys

Primary grouping: `promptRouteKey` (normalized: null/blank -> "unknown"). Each metric row also reports the route
composition -- `selectedPromptRecipeId`, `selectedPromptVersion`, `selectedDataRegime`, `promptSource`,
`fallbackReason`, `legacyFallbackUsed` -- as the uniform value across the group, or null when the group spans
multiple values (common for the heterogeneous "unknown" bucket).

## output metric schema

Per route (`PromptRouteCalibrationMetricRow`): `promptRouteKey`, `selectedPromptRecipeId`,
`selectedPromptVersion`, `selectedDataRegime`, `promptSource`, `fallbackReason`, `legacyFallbackUsed`,
`totalRows`, `reconciledRows`, `unreconciledRows`, `noDecisionRows`, `fallbackRows`, `registryRows`, `liveRows`,
`matchedRows`, `unmatchedRows`, `matchRate`, `averageConfidence`, `averageConfidenceMatched`,
`averageConfidenceUnmatched`.

Summary (`PromptRouteCalibrationMetricSummary`): grand `totalRows`, `reconciledRows`, `unreconciledRows`,
`noDecisionRows`, `unknownRouteRows`, `fallbackRows`, `registryRows`, `liveRows`, `matchedRows`, `unmatchedRows`,
`matchRate`, `averageConfidence`, and the ordered `routes` list (by descending totalRows, then route key).

## metric definitions

- **reconciled (scorable):** a row with BOTH a directional outcome (`resultSide` in home/away/draw) AND a
  directional lean (`leanSide`). Only these are scored.
- **matched:** reconciled AND `leanSide == resultSide` (ordinal). **unmatched:** reconciled AND
  `leanSide != resultSide`. So `matched + unmatched == reconciled`.
- **noDecision:** a directional outcome but NO lean (the run produced a result but no directional read) --
  counted separately, never a miss.
- **unreconciled:** no directional outcome (no outcome recorded, or a non-directional outcome like
  cancelled/postponed) -- never a miss. The partition is clean: `total == reconciled + noDecision +
  unreconciled`.
- **matchRate = matched / reconciled**, or **null** when `reconciled == 0` (never 0/0 -> 0). Unreconciled and
  no-decision rows never deflate it.
- **fallbackRows** = `legacyFallbackUsed == true`. **registryRows** = `promptSource == "registry"`.
  **liveRows** = `promptSource == "live"`.

## matched / unmatched / unreconciled rules

Unreconciled and no-decision rows are excluded from `matched`/`unmatched`/`reconciled` and from the match-rate
denominator -- they are never treated as losses. Only genuinely scorable rows (directional outcome + directional
lean) contribute to matchRate.

## null / unknown route behavior

Missing/blank `promptRouteKey` normalizes to "unknown" and groups together; `unknownRouteRows` on the summary
counts them. The "unknown" bucket's route composition fields collapse to null where they differ across rows
(e.g. mixed live/disabled vs mismatch), and report the uniform value where they agree (e.g. `promptSource =
"live"`).

## confidence averaging behavior

`averageConfidence` is the mean over rows with a numeric `confidence`; rows with null confidence are excluded,
and the average is null when no row in the set has a confidence. `averageConfidenceMatched` /
`averageConfidenceUnmatched` apply the same rule over the matched / unmatched subsets.

## tenant-scope posture

The calculator is a PURE aggregation over rows and performs no tenant filtering itself. Tenant scoping lives
upstream in `PromptRouteCalibrationExporter.ExportAsync(tenantKey)`, which only ever returns one tenant's rows.
A future endpoint/job must obtain rows from the tenant-scoped exporter before calling `Calculate`, so metrics
inherit tenant isolation from the export step.

## buyer-facing impact

None. Internal pure calculator + DTOs; no endpoint, no buyer UX/copy/body, no prompt/template/recipe change, no
confidence/model change. Route metadata + outcome counts only -- no prompt text, prompt bytes, chain-of-thought,
or protocol prose.

## paid-call status

None.

## tests run

- `dotnet build` (DevCore.Api + tests) -> 0 warnings, 0 errors.
- `dotnet test DevCore.Api.Tests --filter PromptRouteCalibrationMetricsTests` -> 11 passed (grouping; reconciled/
  matched/unmatched + matchRate; unreconciled not-a-miss; no-decision excluded; matchRate null when no
  reconciled; unknown normalization; fallback/registry/live counts; confidence averaging + null handling; empty
  input; no-chain-of-thought).
- `dotnet test DevCore.Api.Tests` (full) -> **969 passed, 0 failed** (was 958; +11).
- `pytest tests/test_route_calibration_export.py -q` -> 12 passed (python export path intact).
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- No paid model calls.

## rollback plan

Delete `DevCore.Api/AgentRuns/PromptRouteCalibrationMetrics.cs` and its test. Purely additive (a pure calculator +
DTOs, no DI, no schema/contract/data change), so removal is clean and affects nothing else.

## risks / deferred items

- No HTTP endpoint/job yet (by design) -- the metrics are computed by a pure calculator; a thin tenant-scoped
  endpoint/job that composes exporter + calculator is the natural next slice.
- Outcome coverage depends on `AgentRunOutcome` reconciliation; routes with no reconciled rows report
  `matchRate = null` (correct, not zero).
- The provenance column migration `20260629174632_AddAgentRunPromptRouteProvenance` is still unapplied to live
  SQL (dev container down across prior slices) -- apply on next deploy before running against live data.
- Deliberately conservative: no ROI/CLV/market-edge metrics; matchRate is a lean-vs-result hit rate only, not a
  calibration-curve or profit claim.

## next recommended slice

Thin Tenant-Scoped Calibration Metrics Endpoint or Job v1 (compose exporter + calculator behind a tenant-scoped
internal endpoint/scheduled job), OR Live-Scheduled Starter-Missing Soak v1, OR Broad Cohort Rerun Grouped by
Prompt Recipe v1.

Related: [[platform-side-prompt-provenance-calibration-export-v1]],
[[dotnet-agentrun-prompt-provenance-persistence-v1]], [[prompt-provenance-calibration-export-v1]],
[[default-allowlist-widening-v1]].
