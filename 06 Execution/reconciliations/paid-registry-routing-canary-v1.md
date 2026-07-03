---
title: "Paid Registry Routing Canary v1"
type: "reconciliation"
date: "2026-07-03"
status: "complete -- PASS (registry routing validated end-to-end on a real paid run)"
project: "DAI"
slice: "Paid Registry Routing Canary v1"
repos:
  dai: "unchanged (a923db4)"
  dai-vault: "docs-only"
tags:
  - prompting
  - registry
  - routing
  - canary
related:
  - "02 Platform/decisions/0009-registry-routing-canary-ready.md"
  - "06 Execution/reconciliations/registry-authoritative-routing-canary-v2.md"
  - "06 Execution/reconciliations/source-readiness-preflight-gate-v1.md"
---

# Paid Registry Routing Canary v1 -- PASS

**slice:** run exactly ONE paid registry-authoritative routing canary on a source-readiness-eligible
starter_enriched_market_backed_depth MLB game; validate the full chain end-to-end with byte-identical prompt
behavior + complete provenance. **outcome: PASS.** 1 paid model call; registry routing worked end-to-end; no
fallback; default config unchanged (registry routing re-disabled after the run).

## approval + scope

operator explicitly approved 1 paid registry canary in-thread. scope honored: exactly 1 model call, 1 MLB
game, source-readiness eligible for backed_depth, registry env enabled process-scoped for this run only
(never written to `.env`), v8 cohort untouched (still 8 calls).

## repo / service state

dai `a923db4`, dai-vault `dba580f` (both synced). required commits present: registry canary v2 (a923db4),
source-readiness (e2aaeee), attribution contract (96e1799), residue contract (beed3fc). services: devcore-sql
up; DevCore.Api :5007; agent-service :8000 restarted with `DAI_MLB_REGISTRY_PROMPT_CANARY=1` for the run,
then restarted DEFAULT-OFF after. `.env` never modified (flag absent throughout).

## source-readiness screening (2026-07-03T23:31Z, ~18-27h pre-game)

markets had posted since the ~21:00Z screen. 9 of 10 approved games now ELIGIBLE for
starter_enriched_market_backed_depth (enriched starter + 4-6 book backed_depth); only 823526 MIN@NYY still
starter=missing (Yankees home TBD). eligible: 822882, 823118, 824171, 824903, 824092, 825063, 823932,
824415, 824012.

**selected:** 822882 DET@TEX (6-book market, enriched, Scheduled 07-04T20:05Z, pre-game) -- a fresh game not
used by the prior canaries.

## paid canary run

- **AgentRunId:** c849433e-f36b-1410-8173-00373db4b724
- **gamePk / identity:** 822882 (ExternalGameId 822882, SourceProvider mlb_statsapi) -- matched.
- **game:** Detroit Tigers @ Texas Rangers, 2026-07-04.
- **paid model calls used:** 1 (gpt-4o-mini). lean prose "slight lean toward Texas Rangers (market
  consensus)", confidence 0.75, evidenceRichness 2. (structured leanSide is null on this run -> it would
  settle inconclusive; a directional-read property, orthogonal to the routing/provenance validation.)
- AgentRuns 265 -> 266 (exactly 1 new run).

## registry provenance verification -- ALL PASS

| field | value | pass |
|---|---|:-:|
| observedDataRegime | starter_enriched_market_backed_depth | YES |
| selectedDataRegime | starter_enriched_market_backed_depth | YES |
| selectedPromptPath | registry | YES |
| promptSource | registry | YES |
| recipeId | mlb.pregame.analysis.starter_enriched_market_backed_depth.v1 | YES |
| recipeVersion | v1 | YES |
| assembledHash | 267ca80089fb9f921dde3c356b4f590efec666834c4fb7cee568fbcc7b45e327 (sha256) | YES |
| registryAuthoritativeEnabled | true | YES |
| regimeAllowlisted | true | YES |
| fallbackReason | null | YES |
| fallbackDetail | null | YES |
| attributionStatus | complete | YES |
| livePromptTemplateKey | null | YES |
| observed == selected | yes | YES |
| no fallback to live | confirmed (log: source=registry ... fallback=None) | YES |

`/rows` surfaces it as a real registry-routed row: promptSource=registry, selectedDataRegime populated,
recipeId+version, selectedPromptPath=registry, attributionStatus=complete, promptRouteKey
`mlb.pregame.analysis.starter_enriched_market_backed_depth.v1@v1::starter_enriched_market_backed_depth`
(NOT "unknown"). This is the first real backed_depth ROUTE-attributed row.

byte identity: dry-run tests (registry canary v2) proved the recipe assembles byte-identical to live; on this
run the decision selected registry with fallbackReason=None, which the code path only does after
`registry_text == live_prompt` byte-for-byte -> the model input was byte-identical to the live prompt.

## settlement status

**not final** (07-04T20:05Z, pre-game) -> NOT reconciled. 0 reconciliation writes.
**watch plan:** after DET@TEX is Final (~07-04T23:30Z+), verify StatsAPI final, run reconcile-precheck for
gamePk 822882, and if final + identity-safe + not-already-reconciled settle via the canonical residue
contract (source=statsapi_final, sourceRef="gamePk 822882", notes="Paid Registry Routing Canary v1; <final
score>; via <writer>"), then confirm non-null settlementSource/sourceRef/notes on /rows. NOTE: structured
leanSide null -> this run evaluates inconclusive (no-decision); it validates ROUTING, not a directional row.

## canary decision

**PASS -- registry-authoritative routing works end-to-end on a real run.** source ingredients ->
observedDataRegime -> selectedDataRegime -> registry recipe selected + assembled (byte-identical) ->
complete provenance, with no fallback. It is SAFE to route additional source-readiness-eligible backed_depth
games through registry (identical model bytes; only provenance/route-label differ). This validates the
mechanism; it is NOT a decision to promote registry routing to default or to additional regimes.

## v8 measurement recommendation

registry routing is now proven safe for backed_depth. RECOMMEND (separate operator go): resume the remaining
v8 as a REGISTRY-ROUTED backed_depth cohort while games are source-readiness eligible -- each run gets a real
`selectedDataRegime=starter_enriched_market_backed_depth` ROUTE row (byRoute), identical bytes/decision, and
still feeds the 0.80 confidence bucket. Do NOT enable registry routing in default config; scope the env per
run/session. Alternatively keep v8 live-path (unknown route) -- but registry-routed is strictly more
informative now that it is proven. Either way, 8 v8 calls remain and are NOT spent here.

## safety ledger

paid model calls 1 (the approved canary; v8's 8 remain untouched); new game runs 1 (822882, eligible +
approved); reconciliation writes 0 (not final); DB migrations 0; prompt text 0; live-path default behavior 0
(agent-service restarted default-off after); registry default changed 0 (`.env` never touched; process env
only, reverted); decision behavior 0; buyer-visible output 0; metrics denominator 0; historical rows
backfilled 0; v8 remaining calls spent 0.
