---
title: "Source Readiness Preflight Gate v1"
type: "reconciliation"
date: "2026-07-03"
status: "complete"
project: "DAI"
slice: "Source Readiness Preflight Gate v1"
repos:
  dai: "code + tests (committed local, unpushed)"
  dai-vault: "docs-only"
tags:
  - calibration
  - retrieval
  - readiness
  - spend-protection
related:
  - "02 Platform/decisions/0008-source-readiness-preflight-gate.md"
  - "06 Execution/reconciliations/v8-cohort-execution-canary-halt-v2.md"
---

# Source Readiness Preflight Gate v1

**slice:** prevent paid model calls on games that cannot reach `starter_enriched_market_backed_depth` --
add a read-only, pre-model readiness gate that reuses the SAME retrieval as generation.
**status:** complete 2026-07-03. `dai` code+tests committed local (unpushed); `dai-vault` docs-only.
**verification:** DevCore.Api.Tests 1052/1052; live screening left AgentRuns unchanged (265); 0 model calls.

## canary v2 recap

MIN@NYY (c149433e): observedDataRegime `starter_missing_market_missing`. starter: home Yankees TBD in the
generation endpoint; market: no odds event matched. halted (gate failed), but a paid model call was spent.

## retrieval path inventory

| path | file/class | source api | paid? | writes db? | model? | reusable pre-model |
|---|---|---|:-:|:-:|:-:|:-:|
| free v8 preflight (WRONG endpoint) | statsapi `feed/live` (ad-hoc) | statsapi | free | no | no | -- misled: disagrees w/ generation |
| generation starter | `MlbStarterClient` | statsapi `schedule?hydrate=probablePitcher` + `people/{id}/stats` | free | no | no | YES |
| generation market | `OddsMarketClient` | the-odds-api `/v4/.../odds` (ET-date window, exact-name match) | **PaidExternal** | no | no | YES |
| retrieval orchestrator | `SportsRetriever.RetrieveAsync` | above, via ToolGateway | free+odds | **no** (records on in-memory artifact) | no | **YES -- the reuse core** |
| generation dispatch | `AgentRunService.RunSportsMatchupPipelineAsync` | -- | -- | writes AFTER (controller) | **yes** (AnalyzeAsync) | no |
| readiness gate (NEW) | `GET /source-readiness` -> SportsRetriever + `SourceReadinessClassifier` | same as generation | odds | **no** | **no** | -- |

**Key:** `RetrieveAsync` is step 2 of generation, BEFORE the model call (step 3) and BEFORE persistence
(controller, after return). So it is the exact reusable, write-free, pre-model core.

## read-only reproduction (live, no model calls, no writes)

Root cause of canary v2, confirmed: **MlbStarterClient uses `schedule?hydrate=probablePitcher` and requires
BOTH probables** (Yankees home read TBD -> null starter context -> missing). The v8 free preflight used
`feed/live` (showed Rodon) -- a DIFFERENT endpoint that disagreed. **OddsMarketClient** matches by exact
team name in an ET-date window; books post games at different times.

Screening all 10 approved candidates via the gate at ~2026-07-03T21:00Z (~20-29h pre-game):

| gamePk | game | starter | market | predicted regime | eligible |
|---|---|---|---|---|---|
| 823526 | MIN@NYY | missing (home TBD) | missing | starter_missing_market_missing | NO |
| 822882 | DET@TEX | enriched | missing | starter_enriched_market_missing | NO |
| 823118 | TOR@SEA | enriched | missing | starter_enriched_market_missing | NO |
| 824171 | TB@HOU | enriched | missing | starter_enriched_market_missing | NO |
| 824903 | NYM@ATL | enriched | missing | starter_enriched_market_missing | NO |
| 824092 | PHI@KC | enriched | missing | starter_enriched_market_missing | NO |
| 825063 | MIL@AZ | enriched | missing | starter_enriched_market_missing | NO |
| 823932 | SD@LAD | enriched | missing | starter_enriched_market_missing | NO |
| 824415 | CWS@CLE | enriched | missing | starter_enriched_market_missing | NO |
| 824012 | BOS@LAA | enriched | missing | starter_enriched_market_missing | NO |

**Diagnosis: TIMING.** starters are mostly ready (9/10 enriched); **markets are uniformly not ready** --
the odds feed has no matched event for any 07-04 game this far out. (Note: SD@LAD had a 3-book market last
session ~34h out; missing now -- odds availability is provider/time-variable, not monotonic.) The blocker
is odds-posting time, not team matching or a code defect. AgentRuns unchanged (265) -> screening wrote
nothing.

## source-readiness contract

fields: sourceProvider, externalGameId, game, gameDate, identityStatus, starterReadiness
(level enriched/asymmetric/named/missing + home/away/source/reason), marketReadiness (level
backed_depth/backed/missing + bookCount/consensusSide/reason), predictedObservedDataRegime,
generationEligibleForTargetRegime, targetRegime (`starter_enriched_market_backed_depth`), stopReason,
checkedAtUtc. Eligibility = identity matched AND starter enriched AND market backed_depth. Deterministic,
mirrors the python observed-regime logic so prediction == the stamped observedDataRegime.

## changes made

- `DevCore.Api/AgentRuns/SourceReadiness.cs` (new): contract records + `SourceReadinessClassifier`.
- `DevCore.Api/Controllers/AgentRunsController.cs`: `GET source-readiness` (inject `ISportsRetriever`;
  ephemeral artifact -> RetrieveAsync -> classify; no model, no write). auth-boundary gated as the rest of
  the controller.
- tests: `SourceReadinessClassifierTests` (6: missing/missing, enriched/backed_depth eligible, asymmetric,
  single-book, no-consensus, unmatched-identity) + 2 endpoint tests (no-op retriever -> missing/missing +
  0 AgentRuns + 0 snapshots; invalid date -> 400).

## tests / verification

- `dotnet test DevCore.Api.Tests`: **1052 passed** (1044 + 8). proves classification correctness; endpoint
  is read-only (0 AgentRuns + 0 MarketSnapshotBatches; no model). live: AgentRuns 265 -> 265.
- no retrieval/prompt/decision/buyer/denominator behavior changed; no migration; registry routing still off.

## v8 resume recommendation

**WAIT until closer to first pitch, then screen-then-generate.** the candidate list is still suitable
(9/10 have enriched starters); the only blocker is market timing. do NOT resume now (all 10 ineligible ->
all 8 remaining calls would be wasted on market-missing runs). Resume flow on 2026-07-04: run
`GET /source-readiness` per candidate a few hours before first pitch; generate ONLY games returning
`generationEligibleForTargetRegime=true` (starter enriched + market backed_depth + identity matched);
re-confirm via the attributed canary; settle finals via the residue contract. 8 model calls remain under
the cap. No separate retrieval fix is REQUIRED (the misses are data-timing); a schedule->feed/live starter
fallback is an optional future robustness slice.

## safety ledger

paid model calls 0; paid external source (odds) calls ~10 (read-only screening, same call generation makes);
new game runs 0; reconciliation writes 0; DB migrations/schema 0; prompt text 0; prompt-selection 0; model
input 0; decision behavior 0; buyer-visible output 0; metrics denominator 0; registry routing enabled NO;
historical rows backfilled NO.
