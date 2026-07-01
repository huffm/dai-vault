---
title: "Targeted Live Batch Capture v1"
type: "evidence-report"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Targeted Live Batch Capture v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - source-depth
  - capture
related:
  - "06 Execution/reports/controlled-live-batch-capture-v2.md"
---

# Targeted Live Batch Capture v1 -- Evidence Report

**status:** complete (9-run approved paid batch; both thin regimes hit; pending reconciliation)
**date:** 2026-06-30

## purpose

Run a targeted live MLB batch against the discovered L+2 candidate slate (2026-07-02) before odds post, to
deliberately exercise the under-evidenced allowlisted regimes starter_missing_market_missing and
starter_enriched_market_missing. Validates the lead-time reachability theory from Regime Discovery v1.

## start state

- `dai`: clean, synced, `0f563d6` (0/0). `dai-vault`: clean, synced, `77f8f05` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4 regimes). Route fix v1 present (regime::fallbackReason, line 147).
- devcore-sql up; FastAPI uvicorn :8000 (canary=1, provenance sink); .NET :5215 (dev bypass, tenant 1). Both healthy.

## candidate re-probe (signals unchanged since discovery)

`statsapi schedule date=2026-07-02 hydrate=probablePitcher` + `the-odds-api baseball_mlb/events` (free):

- **Starters unchanged:** 6 candidates still >=1 TBD (starter_missing); 3 still both-announced.
- **Odds horizon still ~1 day:** only the 06-30 slate has odds events; ZERO 07-02/07-03 events -> all 9
  candidates still `market_missing`.
- **Season-quality confirmed:** all 6 starters in the 3 announced games have 2026 pitching stats
  (Jared Jones ERA 5.76, Alan Rangel 4.50, Chase Burns 2.36, Jacob Misiorowski 1.45, Walbert Urena 3.14,
  Bryce Miller 1.97) -> quality present on both sides -> `enriched` (no named degradation, symmetric -> no
  assembly_error risk).

## duplicate check

Re-dedupe by (mlb_statsapi, gamePk) vs AgentRuns: **all 9 candidate gamePks still new** (0 existing runs).

## approval gate

Presented the re-probed candidate table (9 games, expected regimes, allowlist status, dupe), batch size, expected
calls (1/run), est. cost (<$0.025), env vars, exact POST flow, verification plan. **Operator approved all 9.**
No paid calls before approval.

## approved batch (all 9 completed)

| AgentRunKey | runId | game | gamePk | expected regime | actual regime | source |
|---|---|---|---|---|---|---|
| 270021 | e440433e | Marlins @ Rockies | 824335 | starter_missing_market_missing | starter_missing_market_missing | registry |
| 270022 | e540433e | White Sox @ Guardians | 824416 | starter_missing_market_missing | starter_missing_market_missing | registry |
| 270023 | ea40433e | Cardinals @ Braves | 824906 | starter_missing_market_missing | starter_missing_market_missing | registry |
| 270024 | eb40433e | Rays @ Royals | 824093 | starter_missing_market_missing | starter_missing_market_missing | registry |
| 270025 | ef40433e | Tigers @ Rangers | 822884 | starter_missing_market_missing | starter_missing_market_missing | registry |
| 270026 | f640433e | Padres @ Dodgers | 823935 | starter_missing_market_missing | starter_missing_market_missing | registry |
| 270027 | f940433e | Pirates @ Phillies | 823442 | starter_enriched_market_missing | starter_enriched_market_missing | registry |
| 270028 | fc40433e | Reds @ Brewers | 823765 | starter_enriched_market_missing | starter_enriched_market_missing | registry |
| 270029 | 0341433e | Angels @ Mariners | 823119 | starter_enriched_market_missing | starter_enriched_market_missing | registry |

## expected vs actual regimes

**9/9 matched the predicted regime.** The lead-time reachability theory is confirmed end-to-end: probing the L+2
slate before odds post routes future-dated TBD games to starter_missing_market_missing and future-dated
announced games to starter_enriched_market_missing. The 6 starter_missing_market_missing runs produced a NULL
lean (no-decision: with no starters and no market the analyzer offers no directional read -- expected, honest);
the 3 enriched_market_missing runs produced home leans.

## prompt route provenance evidence

All 9 runs persisted non-null `PromptRouteProvenanceJson`; **9/9 registry-authoritative** (promptSource=registry,
recipe `...<regime>.v1@v1`, fallbackReason null, legacyFallbackUsed false). **No fallback occurred** -- symmetric
quality on the enriched games meant the enriched recipe assembled cleanly. `GET /artifact` verified to expose
promptRouteProvenance (e440433e: registry, starter_missing_market_missing.v1; f940433e: registry,
starter_enriched_market_missing.v1).

## metrics before / after

`GET /api/agent-runs/prompt-route-calibration/metrics` (tenant 1):

| metric | before | after |
|---|---|---|
| totalRows | 254 | **263** (+9) |
| reconciledRows | 76 | 76 |
| unreconciledRows | 170 | **179** (+9) |
| noDecisionRows | 8 | 8 |
| matchedRows | 47 | 47 |
| unmatchedRows | 29 | 29 |
| matchRate | 0.6184 | 0.6184 |
| registryRows | 18 | **27** (+9) |
| liveRows | 1 | 1 |
| fallbackRows | 1 | 1 |
| unknownRouteRows | 235 | 235 |

Target routes:

- **starter_missing_market_missing**: 2 -> **8** (+6), all registry, all unreconciled.
- **starter_enriched_market_missing**: 0 -> **3** (NEW route, first-ever rows), all registry, all unreconciled.
- starter_missing_market_backed_depth: still 1 (not targeted). enriched_market_backed_depth: still 15.

All deltas as expected: totalRows +9, unreconciled +9, registry +9; matchRate unchanged (no new outcomes);
live/fallback unchanged (no fallback).

## fallback route-key verification

No fallback occurred this batch, so no new fallback row. The pre-existing fallback route
`starter_enriched_market_backed_depth::assembly_error` remains correctly attributed (total 1, src=live, fb=1) --
not collapsed to `unknown`. Calibration Route Attribution Fix v1 holds.

## paid-call count and cost

**9 paid gpt-4o-mini calls -- one completed analysis per run** (agent log: 9 "mlb analysis response" + 9
"mlb prompt routing decision", **0 retries**). Estimated cost: **< $0.025**.

## buyer-facing impact

**None.** `SportsAnalysisResponse` unchanged; no prompt/recipe/template change; no chain-of-thought exposed.

## default allowlist status

**Unchanged** -- exactly four regimes.

## pending reconciliation notes

All 9 games are on the 2026-07-02 slate (not Final). All 9 have 0 `AgentRunOutcome` rows and
`ExclusionReason = null` (active). **PENDING -- not reconciled.** No outcomes fabricated; no future game
reconciled. The 6 starter_missing_market_missing runs have null leans and would become no-decision rows on
reconciliation (still valuable route-coverage evidence); the 3 enriched_market_missing runs have directional
leans and will produce matched/unmatched once settled.

## risks / deferred items

- 6 of 9 runs are no-decision (null lean) -- route coverage gained, but starter_missing_market_missing will not
  yield a directional match rate; its calibration value is coverage + confidence behavior, not win rate.
- Future-dated capture (2-day-out, no market) is an unusual artifact shape but valid; reconciles by gamePk once
  Final. Watch that these artifacts read sensibly.
- 20 live runs now pending settlement (3 soak + 8 v2 + 9 targeted) -> reconciliation backlog growing.
- starter_missing_market_backed_depth still thin (1) -- needs the narrow L+1 odds-posted-but-starter-TBD window.

## next recommended slice

**Outcome Reconciliation Follow-up v1** (non-paid, time-gated). The routing layer now has live coverage across 4
of 9 regimes (3 of 4 allowlisted have meaningful counts), but the reconciliation backlog is 20 runs. Settling
them converts coverage into performance evidence at zero paid cost and is the prerequisite for any
allowlist-promotion decision. Continue Targeted Live Batch Capture only to chase
starter_missing_market_backed_depth or to grow enriched_market_missing beyond n=3.
