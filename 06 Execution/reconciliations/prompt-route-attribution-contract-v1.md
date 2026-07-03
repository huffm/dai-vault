---
title: "Prompt Route Attribution Contract v1"
type: "reconciliation"
date: "2026-07-03"
status: "complete"
project: "DAI"
slice: "Prompt Route Attribution Contract v1"
repos:
  dai: "code + tests (committed local, unpushed)"
  dai-vault: "docs-only"
tags:
  - calibration
  - attribution
  - observability
  - prompting
related:
  - "02 Platform/decisions/0007-prompt-route-attribution-contract.md"
  - "06 Execution/reconciliations/v8-cohort-execution-canary-halt-v1.md"
---

# Prompt Route Attribution Contract v1

**slice:** make every generated artifact record analyzable regime-level + prompt-path attribution even when
registry-authoritative routing is disabled -- so v8's backed-depth cohort is measurement-grade without a
prompt-selection change. attribution + observability only.
**status:** complete 2026-07-03. `dai` code+tests committed local (unpushed); `dai-vault` docs-only.
**verification:** agent-service pytest 425/425; DevCore.Api.Tests 1044/1044; 0 paid calls / runs / writes.

## root cause (config, not defect)

`registry_prompt_canary.decide_model_prompt`: when the canary is disabled (the default), it returns the
"disabled" decision BEFORE any regime is computed, so `selectedDataRegime` stays None and the pooled tool
(grouping by `selectedDataRegime`) buckets the run `unknown`. Files: `decide_model_prompt` / `_live`
(registry_prompt_canary.py), consumed by `sports_analyzer` (~L984) -> `on_route_decision` ->
`build_route_provenance` (route_provenance.py) -> `X-Prompt-Route-Provenance` header -> C#
`PromptRouteProvenance.TryParseHeader` -> `/rows`. A deterministic regime classifier already existed
(`dataregime.py` + `migration_readiness._starter_state/_market_state`) but was only invoked when routing
was enabled. **Expected config behavior, not a defect** -- but it left live-path artifacts unattributed.

## attribution inventory (before -> after)

| field | producer | live-path before | after |
|---|---|---|---|
| selectedDataRegime | registry selection | null | null (unchanged -- selection only) |
| promptRouteKey (C#) | RouteKey(recipe/regime/fallback) | "unknown" | "unknown" (unchanged) |
| promptSource | decision | "live" | "live" |
| fallbackReason/Detail | decision | "disabled"/null | unchanged |
| recipeId/version/hash | registry selection | null | null (never fabricated) |
| **observedDataRegime** | `observed_data_regime(inputs)` | (absent) | **stamped every run** |
| **selectedPromptPath** | decision | (absent) | **"live"/"registry"/"fallback"** |
| **livePromptTemplateKey** | constant marker | (absent) | **"mlb.matchup.analysis.live"** |
| **attributionStatus/Reason** | decision | (absent) | **partial / complete / unattributed** |

## attribution contract

- **observedDataRegime**: regime the inputs support, deterministic from the same typed evidence the router
  uses. equals selectedDataRegime when routing is on. observability only.
- **selectedDataRegime**: registry SELECTION only -- stays null on live path (not overloaded).
- **selectedPromptPath**: live | registry | fallback.
- **livePromptTemplateKey**: truthful live-template marker; never a registry recipe id; null on registry.
- **attributionStatus**: complete (registry recipe selected) | partial (observed-only live path) |
  unattributed (classification failed). **attributionReason**: explanation.
- **/rows mapping**: all five surfaced additively (trailing-optional, metrics-ignored). pooled tool gains
  additive `byObservedRoute` (keyed on observedDataRegime) beside unchanged `byRoute`.

## changes made

- `services/agent-service/app/services/migration_readiness.py` -- `observed_data_regime()` (pure classifier,
  reuses existing state helpers).
- `.../registry_prompt_canary.py` -- `LIVE_PROMPT_TEMPLATE_KEY`, `_safe_observed_regime` (guarded), compute
  observed once in `decide_model_prompt`, stamp attribution on every branch (`_live` + registry-success).
- `.../app/prompting/contracts.py` + `route_provenance.py` -- 5 additive fields on `PromptRouteDecision` +
  `PromptRouteProvenance`; `build_route_provenance` copies them.
- `.../app/services/pooled_calibration.py` -- additive `byObservedRoute` grouping (no denominator change).
- `platform/dotnet/DevCore.AiClient/PromptRouteProvenance.cs` -- 5 nullable fields (header parses 1:1).
- `platform/dotnet/DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` -- 5 fields on the row, mapped from
  the parsed provenance. RouteKey/selectedDataRegime semantics unchanged.
- tests: python (canary attribution incl. the exact v8 fixture, provenance propagation, pooled byObservedRoute)
  + C# (live-path attribution surfaces on /rows). exact-dict provenance serialization test updated for the
  new keys.

## tests / verification

- `pytest` (agent-service): **425 passed**. proves: disabled live path stamps observedDataRegime for
  backed_depth / market_missing / starter+market missing; prompt bytes unchanged (`prompt == live`); no
  registry selection on live; provenance projection carries attribution; byObservedRoute attributes live rows
  that byRoute calls unknown; denominator unchanged.
- `dotnet test DevCore.Api.Tests`: **1044 passed** (1043 + 1). proves /rows surfaces observed vs selected as
  distinct fields; metrics + buyer-boundary tests unchanged.
- Non-semantic proof: no template/model-input/selection/decision/buyer change; registry routing still
  DEFAULT-OFF; /metrics logic untouched.

## v8 resume decision

**RESUME SAFE.** Observed backed-depth attribution is now available WITHOUT any behavior change or registry
routing. The remaining 9 approved games, generated under the unchanged pipeline, will carry
`observedDataRegime = starter_enriched_market_backed_depth` (attributable via `byObservedRoute`) and land in
the 0.80 bucket. Conditions: (1) still an operator-approved PAID action (the v8 10-game / 10-call approval +
cap stand; 1 call already spent, 9 remain); (2) re-run the canary and confirm the fresh run's `/rows` shows
`observedDataRegime=starter_enriched_market_backed_depth`, `attributionStatus=partial`,
`selectedPromptPath=live` before generating the rest; (3) `selectedDataRegime` will remain null (that is
correct -- routing stays off); attribution is via observedDataRegime. Enabling registry routing is a
SEPARATE approved slice and is NOT required.

## safety ledger

paid calls 0; new game runs 0; reconciliation writes 0; DB migrations/schema 0; prompt text 0;
prompt-selection behavior 0; model input 0; decision behavior 0; buyer-visible output 0; metrics denominator
0; registry-authoritative routing enabled NO; historical rows backfilled NO.
