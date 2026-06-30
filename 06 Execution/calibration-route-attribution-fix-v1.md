# Calibration Route Attribution Fix v1 -- Evidence Report

**status:** complete (code + tests shipped; verified on dev data; not pushed)
**date:** 2026-06-30

## purpose

Fix calibration route attribution for safe live fallbacks whose data regime is known but recipe/version/hash are
null (e.g. `assembly_error`). These runs should not collapse into the generic `unknown` bucket; they get a
fallback-specific route key `selectedDataRegime::fallbackReason` (e.g.
`starter_enriched_market_backed_depth::assembly_error`) while staying honest that `promptSource` was `live`.

This is a route-attribution slice -- not a prompt rewrite, not a registry assembly fix, not a paid-call slice,
not a cohort rerun, not OKF.

## start state

- `dai`: clean, synced, `289777f` (0/0). `dai-vault`: clean, synced, `896b96e` (0/0, pushed previous slice).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged -- exactly four regimes. No paid calls.

## problem statement

The route key was derived only from `recipeId@version::regime`, returning `unknown` whenever any of the three was
null. A safe live fallback caused by `assembly_error` has null recipe/version (assembly raised before a recipe
result existed) but a KNOWN `selectedDataRegime` + `fallbackReason`. So run 260018 (Tigers@Yankees) -- which
failed closed to live for regime `starter_enriched_market_backed_depth` -- bucketed to `unknown`, losing its
intended-regime calibration signal (the enriched route showed n=7 instead of 8; the fallback's incorrect outcome
sat in the `unknown` bucket). Identified by the Registry Assembly Error Diagnostic v1 (Option C).

## route-key rule (3-tier priority)

1. **Registry/recipe route:** `recipeId` + `version` + `regime` all present -> `recipeId@version::regime`
   (unchanged behavior for registry-authored rows).
2. **Fallback route:** else if `regime` + `fallbackReason` present -> `regime::fallbackReason`.
3. **Unknown:** else -> `unknown`.

Null/empty/whitespace is treated as missing; values are trimmed. `promptSource` is NOT part of the key. A
fallback route is never treated as registry-authored.

## code changes (minimal; 2 impls + mirror, no contract/schema change)

- `platform/dotnet/DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` --
  `PromptRouteCalibrationExporter.RouteKey(...)` rewritten to the 3-tier rule with a `Trimmed(...)` helper
  (null/empty/whitespace -> missing).
- `services/agent-service/app/services/route_calibration_export.py` -- `prompt_route_key(...)` rewritten to the
  identical 3-tier rule with a `_trimmed(...)` helper (python mirror kept 1:1 with .NET).

No change to: `PromptRouteProvenance` contract, DB schema, persisted `PromptRouteProvenanceJson`, the metrics
aggregator (`PromptRouteCalibrationMetricsCalculator`), reconciliation, confidence, or any buyer surface. The
metrics aggregator already derives `registryRows`/`liveRows`/`fallbackRows` from `promptSource`/
`legacyFallbackUsed` (not the route key), so only grouping shifts.

## test changes (TDD: red -> green)

- NEW `platform/dotnet/DevCore.Api.Tests/AgentRuns/PromptRouteKeyFallbackTests.cs` (8 tests): registry route
  unchanged; assembly_error fallback -> fallback key; missing regime -> unknown; missing reason -> unknown;
  whitespace-only -> unknown; fallback key trimmed; null provenance -> unknown; and a metrics-level test proving
  the fallback route groups separately, counts as `live` (not registry), keeps its own matched/unmatched, and
  leaves `registryRows`/`liveRows`/`fallbackRows` summary counters unchanged.
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/PromptRouteCalibrationExporterTests.cs` -- added
  `assembly_error_fallback_with_known_regime_exports_fallback_route_key` (durable-provenance integration test).
- `services/agent-service/tests/test_route_calibration_export.py` -- added
  `test_assembly_error_fallback_gets_fallback_route_key` (python mirror). Existing `test_live_fallback_row`
  (a `disabled` fallback with regime=None) still asserts `unknown` and stays green.

Red phase before implementation: the 3 fallback-key assertions failed; the unchanged-behavior tests passed.

## before / after behavior

| run 260018 (Tigers@Yankees, assembly_error) | before | after |
|---|---|---|
| promptSource | live | live (unchanged) |
| selectedDataRegime | starter_enriched_market_backed_depth | same |
| promptRouteKey | `unknown` | `starter_enriched_market_backed_depth::assembly_error` |
| counted as registry? | no | no (still `live`) |

## metrics impact (verified live on dev data, tenant 1)

`GET /api/agent-runs/prompt-route-calibration/metrics`, before -> after this fix:

| metric | before | after |
|---|---|---|
| totalRows | 246 | 246 |
| reconciledRows | 76 | 76 |
| matchedRows | 47 | 47 |
| unmatchedRows | 29 | 29 |
| matchRate | 0.6184 | 0.6184 |
| registryRows | 10 | 10 |
| liveRows | 1 | 1 |
| fallbackRows | 1 | 1 |
| unknownRouteRows | 236 | **235** |

New route row: `starter_enriched_market_backed_depth::assembly_error` -- total 1, reconciled 1, matched 0,
unmatched 1, matchRate 0, promptSource live, registryRows 0, liveRows 1, fallbackRows 1, fallbackReason
assembly_error. The `unknown` route dropped by exactly that one row (236->235; its reconciled 69->68,
unmatched 28->27). Only grouping changed; no count was created or destroyed.

## paid-call status

**None.** No model calls. Unit/integration tests + a non-paid metrics endpoint read.

## buyer-facing impact

**None.** Only the calibration export/metrics route key changed. `SportsAnalysisResponse` and the artifact
endpoint are untouched; no prompt text, chain-of-thought, or prose exposed.

## default allowlist status

**Unchanged** -- exactly four regimes.

## risks / deferred items

- Fallback keys now appear in the `routes[]` list. Any downstream consumer that assumed every non-`unknown`
  route is registry-authored must read `promptSource`/`fallbackReason` (already present per row) -- the metrics
  summary counters are unchanged, so aggregate dashboards are unaffected.
- The assembly_error detail (which slot failed) is still not persisted (`PromptRouteDecision` is frozen) --
  carried over from the diagnostic as a separate future enhancement; not in scope here.
- Other fallback reasons (`mismatch`, `regime_not_allowlisted`) with a known regime will now also get a
  fallback key instead of `unknown` -- intended and more honest, but worth noting it widens the route list.

## next recommended slice

**Outcome Reconciliation Follow-up v1** (non-paid, time-gated) -- reconcile the 3 starter-missing soak games
once Final (824338 + 825066 settle 06-30 night; 824818 on 07-01), populating the two `starter_missing` registry
routes' match rates. This is the next concrete non-paid step now that attribution is clean.
