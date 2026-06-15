# Probe Fallback Catalog v1

**date:** 2026-06-15
**status:** implemented as a dormant backend catalog plus docs. no probe execution, no tool gateway call, no model call, no generation, no reconciliation, no gate activation, no prompt change, no dto/api/frontend change, no persistence, no migration.
**classification:** dormant source/tool option catalog.

## Problem statement

Perceive Fulfillment and Question-to-Probe Handoff Contract v1 made the Perceive boundary legible, but the later Probe repair path still lacked a stable menu of approved source/tool options. Probe should know which primary, fallback, or proxy source options may later repair a targeted gap without gaining retrieve power itself.

This slice builds the menu, not the machinery.

## Existing policy and tooling audit

1. **Source/tool policy objects already exist:** `ToolRegistry` defines registered tool ids, allowed protocol nodes, transport, cache, and cost class. `ProtocolToolAccessPolicy` enforces station/tool compatibility. `ProtocolRegistry` keeps `interrogate.probe` on `NoTools`.
2. **Dormant Probe refresh tooling already exists:** `ProbeRefreshDecisionService`, `ProbeRefreshAuthorizationService`, `ProbeRefreshExecutor`, and the larger probe-refresh chain exist. They remain dormant; the executor is disabled by default.
3. **Approved/referenced sports retrieve tools:** `market.football.spread`, `market.basketball.spread`, `schedule.basketball.rest_context`, `pitching.mlb.probable_starters`, and `market.sharp_public.split`. `schedule.matchup_dates` is a reference tool, not a Probe fallback. `analysis.sports.matchup_read` is the analyzer model call, not a Probe fallback.
4. **Known primary source groups:** `identity_schedule`, `market_odds`, `starting_pitching`, `rest_travel`, and `market_movement` have current primary paths for at least one routed sport.
5. **Known fallback/proxy groups:** `identity_schedule` for basketball has `schedule.basketball.rest_context` only as a proxy to confirm schedule context, not as stable odds identity. No live fallback source is activated.
6. **Proxy-only groups:** `fallback_proxy` is a taxonomy bucket, not a source. It has no approved direct path.
7. **No approved path today:** `bullpen_availability`, `lineup_injury`, `team_form`, and `weather_park` are future candidates. MLB `market_odds`, MLB `rest_travel`, and MLB `market_movement` also have no approved current path.
8. **ProbeRequest shape:** no code change needed. `ProbeRequest` remains the singular structured Probe handoff.
9. **Pure implementation viability:** yes. A static catalog can list approved options without calling any runtime service.
10. **Runtime wiring requirement:** none. If wiring were required, code would have been deferred.

## Principal engineer review

1. **Smallest useful catalog worth coding now:** a static source-group plus sport catalog that returns primary/fallback/proxy tool ids and source-policy flags. It is useful because Perceive fulfillment can later ask what groups have approved primary or fallback paths, without resolving tools ad hoc.
2. **Docs-only remains:** Question semantics, fallback ranking, tenant-specific permissions, rate-limit policy, observability, and live Probe execution stay docs-only/deferred.
3. **Existing tool access policy coverage:** yes. `ProtocolToolAccessPolicy` and `ProbeRefreshAuthorizationService` already cover authorization. The catalog is not an authority and does not widen permissions.
4. **Avoiding ProbeRequest duplication:** the catalog does not accept or return `ProbeRequest`; it only maps source groups to tool options.
5. **Avoiding live orchestration:** no DI registration, no endpoint, no controller/composer call, no gateway dependency, no async methods, and no executor call.
6. **Preserving tenant/tool safety:** tenant safety remains with future orchestrator/gateway context; tool safety remains with `ToolRegistry` and `ProtocolToolAccessPolicy`. This catalog cannot bypass either.
7. **Dangerous early activation:** letting `interrogate.probe` call retrieve tools, treating future candidates as approved sources, running probes without tenant permission/rate limits, or using proxies as if they were primary evidence.
8. **Overengineering avoided:** no fallback executor, no new Probe handoff object, no Question object, no source integration, no runtime flag, no persistence, no migration, no buyer surface.

## Catalog vocabulary

- `SourceGroup`: one of the Source Group Taxonomy v1 groups.
- `Sport`: current implementation uses competition codes (`mlb`, `nba`, `nfl`, `ncaaf`, `ncaamb`).
- `PrimaryToolId`: the current approved primary tool for this source group and sport, if one exists.
- `FallbackToolIds`: approved same-purpose fallbacks. None are currently approved.
- `ProxyToolIds`: tools that may provide partial context but must not be treated as primary evidence.
- `IsSupported`: true only when a current registered path exists.
- `RequiresPaidProvider` / `RequiresKey`: source policy flags.
- `CanRunPreGame` / `CanRunInGame` / `CanRunPostGame`: v1 execution window metadata. Current supported options are pregame-only.
- `ReliabilityTier`: `high`, `medium`, `proxy`, `future`, or `none`.
- `Status`: `supported`, `unsupported`, `unavailable`, or `future_candidate`.

## Source group mapping

| source group | current supported sports | primary tool(s) | fallback/proxy | current status |
|---|---|---|---|---|
| identity_schedule | MLB, NBA, NFL, NCAAF, NCAAMB | MLB: `pitching.mlb.probable_starters`; basketball: `market.basketball.spread`; football: `market.football.spread` | basketball proxy: `schedule.basketball.rest_context` | supported where listed |
| market_odds | NBA, NFL, NCAAF, NCAAMB | basketball: `market.basketball.spread`; football: `market.football.spread` | none | MLB is future candidate only |
| starting_pitching | MLB | `pitching.mlb.probable_starters` | none | supported for MLB |
| bullpen_availability | none | none | none | future candidate |
| lineup_injury | none | none | none | future candidate |
| team_form | none | none | none | future candidate; model priors are not Probe fallback |
| rest_travel | NBA, NCAAMB | `schedule.basketball.rest_context` | none | supported for basketball only |
| weather_park | none | none | none | future candidate |
| market_movement | NBA, NFL, NCAAF, NCAAMB | `market.sharp_public.split` | none | supported where `sharp_public` is expected |
| fallback_proxy | none | none | none | unsupported taxonomy bucket |

## MLB current-state mapping

| source group | status | primary | fallback | proxy | notes |
|---|---|---|---|---|---|
| identity_schedule | supported | `pitching.mlb.probable_starters` | none | none | statsapi gamePk/schedule identity comes through probable-starters grounding when matched |
| market_odds | future_candidate | none | none | none | no approved current MLB market retrieve path |
| starting_pitching | supported | `pitching.mlb.probable_starters` | none | none | starters may be unavailable before announcement |
| bullpen_availability | future_candidate | none | none | none | no approved bullpen source |
| lineup_injury | future_candidate | none | none | none | no approved lineup/injury source |
| team_form | future_candidate | none | none | none | model priors are not a Probe fallback |
| rest_travel | future_candidate | none | none | none | no approved MLB rest/travel source |
| weather_park | future_candidate | none | none | none | no approved weather/park-factor source |
| market_movement | future_candidate | none | none | none | `sharp_public` is not an expected MLB signal today |
| fallback_proxy | unsupported | none | none | none | taxonomy bucket only |

## Supported now

- MLB: `identity_schedule`, `starting_pitching`.
- NBA: `identity_schedule`, `market_odds`, `rest_travel`, `market_movement`.
- NFL/NCAAF smoke paths: `identity_schedule`, `market_odds`, `market_movement`.
- NCAAMB smoke path: `identity_schedule`, `market_odds`, `rest_travel`, `market_movement`.

This is source/tool support only. It does not change buyer readiness.

## Unsupported or future

- `bullpen_availability`
- `lineup_injury`
- `team_form`
- `weather_park`
- `fallback_proxy`
- MLB `market_odds`
- MLB `rest_travel`
- MLB `market_movement`

These must not be treated as approved repair paths until a source integration and policy review land.

## Question-to-Probe handoff support

Question remains docs-only/model-heavy. Later, a structured Question trace may say a decision needs a specific source group. This catalog can then answer which primary, fallback, or proxy tool options are approved for that group and sport. It does not create the Question object and does not create a new Probe request.

## PerceiveFulfillmentPolicy support

`PerceiveFulfillmentPolicy` currently accepts explicit `primaryFulfillmentGroups` and `probeFallbackGroups`. A later dormant adapter can read this catalog to derive those group lists. That adapter was not built here because this slice is the menu, not the machinery.

## Why Probe still does not execute

`interrogate.probe` still has no allowed tools. `ProbeRequest` still recommends investigation only. The catalog does not authorize tools, does not call `ToolGateway`, and does not touch `ProbeRefreshExecutor`. Any future execution still needs explicit orchestration, authorization at `platform.retrieve`, tenant context, rate limits, and observability.

## Activation prerequisites

- approved source/tool policy.
- tool gateway wiring from a dedicated orchestrator.
- tenant-scoped permissions.
- rate limits.
- observability.
- calibration impact review.

## Deferred

- live Probe execution.
- new source integrations.
- WNBA support.
- prompt changes.
- buyer-facing display.
- fallback ranking.
- tenant-specific source policy.
- source freshness and stale-data rules.

## Code and tests

Code added:

- `platform/dotnet/DevCore.Api/AgentRuns/ProbeFallbackCatalog.cs`
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/ProbeFallbackCatalogTests.cs`

Tests cover:

- known source group returns expected primary policy.
- unsupported group returns unsupported safely.
- MLB `starting_pitching` has a supported primary path.
- MLB `market_odds` accurately shows no approved current path.
- MLB catalog covers all source groups.
- catalog has no gateway/tool-execution dependency.
- catalog does not accept or return `ProbeRequest`.

## Verification

- `ProbeFallbackCatalogTests`: 7/7 passed.
- `PerceiveFulfillmentPolicyTests | SourceSignalTaxonomyTests | ProbeRequestTests`: 31/31 passed.
- full `DevCore.Api.Tests`: 696/696 passed.

No model calls, no sports generation, no reconciliation, no migrations, and no runtime writes.

## Recommended next slice

**Probe Fallback Catalog Integration Readiness v1**: docs-first or pure-code adapter that derives `primaryFulfillmentGroups` and `probeFallbackGroups` from the catalog for a given sport, still without executing Probe. Do not proceed to live execution until tenant/tool authorization, rate limits, and observability are designed.
