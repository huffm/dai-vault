# MLB Game Identity Capture v1

**date:** 2026-06-11
**status:** runtime capture hardening. narrow platform change + tests in `dai`; report/handoff/ledger in `dai-vault`. no migration, no artifact-contract (OutputJson) change, no buyer-facing change, no matcher, no settlement provider, no taxonomy change, no prompt/parser/confidence/posture/lean/model/cost change.
**scope:** capture stable MLB game identity from statsapi at generation time and persist it through the existing AgentRun identity columns, closing the buyer-ready-sport identity gap left by Stable Game Identity Capture v1.

## Purpose

Stable Game Identity Capture v1 (2026-06-11) captured identity only on odds-market-grounded paths; MLB runs persisted null identity because the MLB pipeline has no odds-market call. MLB is buyer-ready and in season, so every ungrounded MLB run was a permanently fallback-only reconciliation case. This slice captures identity from the provider that actually grounds MLB runs -- statsapi.mlb.com -- using the same contract and the same AgentRun columns. **Both buyer-ready sports (NBA, MLB) now have generation-time identity coverage.**

## Premise verified in code

- the statsapi `/api/v1/schedule` response carries `gamePk` (stable provider event id) and `gameDate` (utc scheduled start) per game, but `MlbScheduleGame` mapped neither -- identity died at the DTO boundary.
- the game match in `MlbStarterClient.FetchStartersAsync` happens before the starter check, so a game can be confidently provider-matched even when probable starters are unannounced (which previously returned null and lost everything).
- `GetStartersAsync` returned only the AiClient `MlbStarterContext` (analyzer payload type -- must not be extended).

## Design

- `MlbStarterGrounding(MlbStarterContext? Context, GameIdentityContext? GameIdentity)` mirrors the market grounding pattern; the AiClient context type and the FastAPI payload are unchanged.
- identity is built at game-match time in `MlbStarterClient`, from the provider's own fields: `SourceProvider = "mlb_statsapi"` (new `GameIdentitySources.MlbStatsApi`, consistent with the existing availability source label), `ExternalGameId = gamePk` as invariant string, `ScheduledStartUtc = gameDate` as utc, `Season` via the existing `GameIdentityDerivation.DeriveSeason("baseball", ...)` (eastern calendar year, e.g. "2026"), team refs as ascii slugs of the provider's home/away names (provider orientation, never the caller's input).
- **identity does not depend on starter announcement** -- a deliberate, spec-mandated divergence from the odds path (which captures only on full grounding): the match is confident as soon as the provider game is found; starter announcement gates only the `starting_pitching` signal. a matched-but-unannounced run degrades to priors-only AND stays reconcilable.
- all-or-nothing conservatism: a matched game missing `gamePk` or `gameDate` carries no identity; pre-match failure paths (network, http error, empty schedule, no match) carry none. identity is never inferred from display names alone -- it always echoes the fields of a provider-matched game, so identity fidelity equals signal-grounding fidelity (the match logic itself, exact-first/contains-fallback, was not changed).
- threading reuses Stable Game Identity Capture v1 end to end: `SportsRetriever` unpacks the grounding into `SportsRetrievalOutput.GameIdentity`; the existing `AgentRunExecution` wrapper, controller persistence (`ApplyGameIdentity`, success + analyze-failure paths), artifact inspection projection, and the existing six AgentRun columns handle the rest with zero changes.

## Changes made

backend (`dai`):
- `DevCore.Api/Sports/GameIdentity.cs`: `GameIdentitySources.MlbStatsApi`; `MlbStarterGrounding` record.
- `DevCore.Api/Sports/MlbStarterClient.cs`: `MlbScheduleGame` DTO gains `GamePk`/`GameDate`; `GetStartersAsync`/`FetchStartersAsync` return `MlbStarterGrounding`; identity built at match time; cache stores the grounding.
- `DevCore.Api/Tools/Handlers/RetrieveSignalHandlers.cs` + `Tools/ToolGatewayServiceCollectionExtensions.cs`: mlb tool output retyped to the grounding.
- `DevCore.Api/AgentRuns/SportsRetriever.cs`: mlb branch unpacks context + identity.

tests (`dai`, red observed first):
- `SportsRetrieverTests`: resumed from the protected red-phase diff -- four MLB identity tests (capture from statsapi; capture even when starters unannounced; null when no game matches; null when provider omits gamePk/gameDate) + `BuildMlbScheduleResponse` fixture; red was observed for the expected production-gap reason (2 failed on null identity, guards green, no syntax repair needed); mlb handler registration retyped.
- `ToolGatewayRetrieveParityTests`: keyed-resolution assertion, invoke test, and registration retyped to the grounding (now pins the production registration type).

## What was deliberately NOT changed

- `ProbeRefreshExecutor`'s mlb arm still invokes the old `MlbStarterContext?` output type -- same dormant type drift as its market arms (created by the prior slice's retyping). per the slice rule it is repaired only on compile failure; it compiles (generic invoke, no compile-time registration check) and the chain is disabled-by-default with fake-gateway tests, so it was left untouched. the drift now covers all three retyped arms and should be repaired in one pass before any probe-refresh activation (already a gate in ledger entries 2/13).
- no migration: the six existing AgentRun identity columns from `20260611152737_AddAgentRunGameIdentity` carry mlb identity unchanged.
- matcher: deferred. settlement-status taxonomy: deferred. settlement provider and scheduled settlement: deferred. buyer-visible track record: deferred. this slice adds run-row metadata only, not buyer output.

## Verification

- skills gate run (dai-skill-router); bounded ladder per dai-test-discipline: SportsRetrieverTests only for red (2 failed for the expected reason, 9 green) and green (11/11); identity-adjacent suites (GameIdentityDerivation, AgentRunService, AgentRunsController, ToolGatewayMarketSpread, ToolGatewayRetrieveParity) 63/63.
- final verification (declared): full .NET suite 637 passed / 0 failed (net +3 vs prior 634: four new tests, one repurposed); full solution build succeeded; `ng test` 47/47; `ng build` exit 0; zero diffs under `apps/` and `services/`; no new migration; `git diff --check` clean; added-line ascii scan clean.
- OutputJson contract: the existing no-identity-in-OutputJson integration test remains green (in the 637).

## Recommended next work

- Outcome Reconciliation Runtime v1 (remainder): the matcher joining settled games to runs on `(SourceProvider, ExternalGameId)`, expanded settlement-status taxonomy, thin manual/import settlement -- still provider-agnostic. mlb settlement can come from the same statsapi (gamePk join), which is why this capture used it.
- begin manual Stage 0 reconciliation on settled NBA/MLB runs via the existing `/outcome` endpoint.
- probe-refresh executor arm-type repair (one pass, all three arms) before any activation -- dormant, not urgent.
