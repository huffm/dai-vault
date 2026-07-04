---
title: "Registry-Routed v8 Backed-Depth Cohort Resume v1"
type: "reconciliation"
date: "2026-07-04"
status: "complete (generation) -- PASS 7/7 registry provenance; settlement pending finality"
project: "DAI"
slice: "Registry-Routed v8 Backed-Depth Cohort Resume"
repos:
  dai: "unchanged (a923db4)"
  dai-vault: "docs-only"
tags:
  - prompting
  - registry
  - routing
  - calibration
  - cohort
related:
  - "06 Execution/reconciliations/paid-registry-routing-canary-v1.md"
  - "02 Platform/decisions/0009-registry-routing-canary-ready.md"
  - "06 Execution/reconciliations/source-readiness-preflight-gate-v1.md"
---

# Registry-Routed v8 Backed-Depth Cohort Resume v1 -- PASS 7/7

**slice:** resume the remaining v8 cohort via registry-authoritative routing for source-readiness-eligible
backed_depth games, under the approved cap, byte-identical prompt behavior + complete route provenance.
**status:** complete 2026-07-04T00:3xZ. **7 paid model calls, 7/7 registry provenance PASS.** settlement
pending finality (games 07-04/07-05). `dai` unchanged; `dai-vault` docs-only.

## approval + scope

operator explicitly approved: 7 gamePks (823118, 824415, 824171, 824903, 824092, 824012, 825063), max 7
paid calls, cumulative v8 cap 10, do NOT spend the 8th remaining call or invent a replacement, registry env
process-scoped only (never `.env`), agent-service default-off immediately after, stop on any provenance
failure, no replacements/retries, no settlement until finality.

## repo / service state

dai `a923db4`, dai-vault `1fcfdcb` (both synced). services: devcore-sql up; DevCore.Api :5007; agent-service
restarted with `DAI_MLB_REGISTRY_PROMPT_CANARY=1` PROCESS-SCOPED for the generation window, then restarted
DEFAULT-OFF immediately after. `.env` registry flag absent throughout (default-off preserved).

## prior v8 runs (do-not-regenerate)

823932/be49433e (SD@LAD, live, canary v1), 823526/c149433e (MIN@NYY, live, canary v2 miss),
822882/c849433e (DET@TEX, registry, paid canary v1). 3 calls spent before this slice.

## source-readiness screening (2026-07-04T00:24Z, all pre-game)

all 7 remaining approved candidates ELIGIBLE + pre-game for starter_enriched_market_backed_depth (enriched
starter + 5-7 book backed_depth market). none skipped.

## registry-routed generation -- 7/7 PASS

| # | gamePk | game | AgentRunId | promptSource | selectedDataRegime | recipe@ver | hash8 | lean | conf | books | attr |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 823118 | TOR@SEA | cb49433e | registry | starter_enriched_market_backed_depth | ...backed_depth.v1 | 5e95a65c | home | 0.80 | 6 | complete |
| 2 | 824415 | CWS@CLE | cd49433e | registry | starter_enriched_market_backed_depth | ...backed_depth.v1 | dae2d0a3 | home | 0.80 | 7 | complete |
| 3 | 824171 | TB@HOU | d049433e | registry | starter_enriched_market_backed_depth | ...backed_depth.v1 | 2250e249 | home | 0.75 | 6 | complete |
| 4 | 824903 | NYM@ATL | d649433e | registry | starter_enriched_market_backed_depth | ...backed_depth.v1 | fc9f3f1d | home | 0.80 | 7 | complete |
| 5 | 824092 | PHI@KC | d949433e | registry | starter_enriched_market_backed_depth | ...backed_depth.v1 | e0fde916 | away | 0.75 | 7 | complete |
| 6 | 824012 | BOS@LAA | dc49433e | registry | starter_enriched_market_backed_depth | ...backed_depth.v1 | cfebdb22 | away | 0.75 | 5 | complete |
| 7 | 825063 | MIL@AZ | dd49433e | registry | starter_enriched_market_backed_depth | ...backed_depth.v1 | 2689baf8 | away | 0.75 | 5 | complete |

every run: promptSource=registry, selectedDataRegime==observedDataRegime==starter_enriched_market_backed_depth,
recipeId=mlb.pregame.analysis.starter_enriched_market_backed_depth.v1 @ v1, assembledHash 64-hex sha256,
registryAuthoritativeEnabled+regimeAllowlisted=true, fallbackReason/Detail null, attributionStatus=complete,
livePromptTemplateKey null, externalGameId==gamePk, NO fallback to live. per-run provenance gate passed for
all 7 (would have aborted the cohort on any failure). /rows surfaces all 7 as registry ROUTE rows.

## cohort summary

- paid model calls this slice: **7**; cumulative v8: **10** (3 prior + 7); remaining cap: **0** (8th
  intentionally unspent -- only 7 candidates, no replacement).
- generated gamePks: all 7 approved. skipped: none.
- **route/provenance: 7/7 PASS registry.** unlike the earlier live-path canaries, all 7 carry a structured
  lean -> these are DIRECTIONAL backed_depth ROUTE rows (not no-decision).
- lean distribution: home 4 (823118, 824415, 824171, 824903), away 3 (824092, 824012, 825063).
- confidence distribution: 0.80 x3 (823118, 824415, 824903), 0.75 x4 (824171, 824092, 824012, 825063).
- market book counts: 5-7. valid for backed_depth measurement.

## settlement status -- watch plan (NOT final)

all 7 games are pre-game (first pitch 07-04T20:10Z through 07-05T01:40Z). **0 reconciliation writes.**
after each is Final (~07-05 early UTC): StatsAPI-confirm final, reconcile-precheck per gamePk, settle only if
final + active + identity-safe + not-already-reconciled via the canonical residue contract (source=
statsapi_final, sourceRef="gamePk <pk>", notes="Registry-Routed v8 Backed-Depth Cohort; <final score>; via
<writer>"), verify non-null settlementSource/sourceRef/notes on /rows + no thin residue.

## pooled reassessment -- PENDING FINALITY

no settlement occurred yet -> not run (no fabricated conclusions). once the 7 settle, expect +7 directional
starter_enriched_market_backed_depth ROUTE rows (byRoute + byObservedRoute + by recipe/version) and +3 to the
0.80 confidence bucket / +4 to 0.75 -- to be measured then. conclusionsAllowed re-check after settlement.

## safety ledger

paid model calls 7 (approved cap 7; cumulative v8 = 10 total budget); new game runs 7; reconciliation writes
0 (not final); DB migrations 0; prompt text 0; default routing changed 0; registry default changed 0 (`.env`
untouched; process env only, reverted default-off after generation); decision behavior 0; buyer-visible 0;
metrics denominator 0; historical rows backfilled 0; unapproved games generated 0; 8th call unspent.
