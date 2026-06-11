# Stable Game Identity Capture v1

**date:** 2026-06-11
**status:** runtime capture / persistence hardening. narrow platform change + one EF migration + tests in `dai`; report/handoff/ledger in `dai-vault`. no artifact-contract (OutputJson) change, no buyer-facing change, no matcher, no settlement provider, no outcome-status taxonomy change, no prompt/parser/confidence/posture/lean/model/cost change.
**scope:** capture stable provider event identity at generation time and persist it as AgentRun run-row metadata, so future outcome reconciliation can join runs to settled games on a canonical key instead of fragile display names.

## Purpose

Outcome Reconciliation Contract v1 (2026-06-09) specified the canonical match key `(sourceProvider, externalGameId)` plus `scheduledStartUtc`, `season`, and normalized team refs, captured on the run at generation time. Nothing was implemented; every run generated since persists no stable identity and is reconcilable only by display-name matching, which the contract rules out for automation. This slice implements the capture half of Outcome Reconciliation Runtime v1. The matcher, taxonomy expansion, and any settlement path remain deferred.

Architectural reason: generated decision artifacts must become future calibration samples. Without a stable event identity on the run row, the calibration loop (ledger entry 12) cannot accumulate reliable evidence, and buyer-visible track-record claims stay unfounded.

## Scope

- six nullable identity columns on `AgentRun`: `SourceProvider`, `ExternalGameId`, `ScheduledStartUtc` (datetimeoffset), `Season`, `HomeTeamRef`, `AwayTeamRef`, plus a nonclustered index on `(SourceProvider, ExternalGameId)` (the canonical match key). one EF migration.
- identity built in `OddsMarketClient` from the matched odds-api event (`id` and `commence_time` were already parsed and previously dropped), threaded through retrieval and the pipeline into controller persistence.
- deterministic derivation: season from sport family + eastern-time scheduled start; team refs as lowercase ascii slugs of the provider's own team names.
- identity surfaced on the artifact inspection endpoint (`AgentRunArtifactDto`) only, read from run-row columns.
- TDD throughout; 23 new tests.

## Non-goals

- no matcher, no automated or manual settlement runtime, no settlement provider choice, no scheduled job.
- no outcome status taxonomy change; `RunEvaluator` untouched.
- no OutputJson artifact contract change (guarded by an explicit test).
- no buyer-facing UI change: zero diffs under `apps/sports-app` and `services/agent-service`; buyer-visible track record stays deferred until reconciliation produces evidence.
- no prompt/parser/confidence/posture/lean/signal-quality/model/cost change; no probe-refresh; no source expansion.
- no pricing/Stripe/auth/tenant/dashboard/deployment change; no Postgres; no Jera change. no push.

## Premise verification (code, not assumption)

- `OddsApiOddsEvent` in `OddsMarketClient.cs` and `OddsApiEvent` in `OddsScheduleClient.cs` both parse `id` and `commence_time`; before this slice only the formatted spread strings and an eastern date string survived. confirmed by grep: `ExternalGameId|SourceProvider|ScheduledStartUtc` had zero hits across `platform/dotnet`.
- `AgentRun` carried promoted `Competition`/`GameDate` (date-only)/`LeanSide` but no event identity, no scheduled start time, no team refs.
- the run row is created and updated in `AgentRunsController.Create`; the pipeline result (`AgentRunExecutionResult`) is serialized wholesale to `OutputJson`, so identity must not ride on it.

## Design decisions

- **identity comes from the event that grounds the market signal.** the market spread fetch already performs a confident event match (team-name equality against the provider's own event, either orientation). identity is built only at that point, from the provider's event fields. if no event is matched, every identity field stays null -- identity is never inferred or fabricated from input display names.
- **grounding records, not context mutation.** the market tools now return `FootballMarketGrounding` / `BasketballMarketGrounding` = (prompt-facing context, game identity). the `FootballMarketContext`/`BasketballMarketContext` records in DevCore.AiClient are unchanged because they are part of the analyzer request payload; the FastAPI surface sees no difference.
- **run-row metadata, not artifact contract.** identity rides `SportsRetrievalOutput.GameIdentity` -> `AgentRunExecution` (a new service-boundary wrapper: result + identity) -> controller persistence. `SportsComposer` never projects it, so `OutputJson` is byte-identical in shape; an integration test asserts the serialized artifact contains no identity properties.
- **failure path keeps identity.** identity is known once retrieve completes, before analyze. `AnalysisPipelineException` now carries it, so failed runs persist identity too (failed runs evaluate inconclusive, but the row keeps its audit identity).
- **season derivation** (`GameIdentityDerivation.DeriveSeason`): eastern-time view of the scheduled start, consistent with the odds clients' date handling. baseball (single-year): eastern calendar year, ex `2026`. basketball/football (cross-year): labelled by start year, ex `2025-26`; august or later starts that season, january-july belongs to the prior start year. unknown families default to single-year; new cross-year families must be added explicitly.
- **team refs** (`GameIdentityDerivation.NormalizeTeamRef`): lowercase ascii slugs of the provider's matched event names (ex `boston-celtics`, `st-louis-cardinals`). normalization of an already-matched name, never a matching mechanism. non-ascii letters are treated as separators, keeping refs deterministic and ascii.

## Known limitations (documented, deliberate)

- **MLB runs persist null identity today.** the mlb pipeline grounds only `starting_pitching` (statsapi) and has no odds-market path, so no odds event identity exists. capturing mlb identity from statsapi's `gamePk` (provider `mlb_statsapi`) is the named follow-up -- it must come from the provider that grounded the signals, per the contract. until then, in-season mlb runs remain fallback-only, which is honest but costs calibration samples for one of the two buyer-ready sports.
- **identity is captured only when the market signal fully grounds.** an event matched but with no bookmaker/spread returns null grounding (unchanged failure semantics). capturing identity from a matched-but-spreadless event is a possible refinement, not done to keep this slice's null-return branches untouched.
- the `(SourceProvider, ExternalGameId)` index is in place but nothing joins on it yet; that is the matcher slice.

## Changes made

backend (`dai`):
- `DevCore.Api/Sports/GameIdentity.cs` (new): `GameIdentityContext`, `GameIdentitySources`, `FootballMarketGrounding`, `BasketballMarketGrounding`, `GameIdentityDerivation` (season + team-ref rules).
- `DevCore.Api/Sports/OddsMarketClient.cs`: spread methods return grounding records; identity built from the matched event; internal `SpreadMarketResult` carries it.
- `DevCore.Api/Tools/Handlers/MarketSpreadHandlers.cs` + `Tools/ToolGatewayServiceCollectionExtensions.cs`: market tool output types are the grounding records.
- `DevCore.Api/AgentRuns/SportsRetriever.cs`: unpacks groundings; threads identity into `SportsRetrievalOutput`.
- `DevCore.Api/AgentRuns/SportsRetrievalOutput.cs`: optional `GameIdentity` member (never serialized to OutputJson).
- `DevCore.Api/AgentRuns/IAgentRunService.cs`: new `AgentRunExecution(Result, GameIdentity)` wrapper; `ExecuteAsync` returns it.
- `DevCore.Api/AgentRuns/AgentRunService.cs`: captures identity after retrieve; returns it on success; attaches it to `AnalysisPipelineException` on analyze failure.
- `DevCore.Api/AgentRuns/PipelineModels.cs`: `AnalysisPipelineException.GameIdentity`.
- `DevCore.Api/Controllers/AgentRunsController.cs`: `ApplyGameIdentity` on success and analyze-failure paths; artifact inspection endpoint projects identity from run columns.
- `DevCore.Api/AgentRuns/AgentRunContracts.cs`: `AgentRunArtifactDto` gains six optional identity fields (inspection surface only).
- `DevCore.Domain/Agentic/AgentRun.cs` + `DevCore.Data/AppDbContext.cs`: six nullable columns, max lengths, match-key index.
- `DevCore.Data/Migrations/20260611152737_AddAgentRunGameIdentity`: additive columns + index; clean down.

tests (`dai`, test-first):
- `DevCore.Api.Tests/Sports/GameIdentityDerivationTests.cs` (new, 13): season per family incl. eastern-boundary and unknown-family default; team-ref slug cases.
- `SportsRetrieverTests` (4 new): football + basketball capture end to end through the real gateway with fake http; null when no event matches; null for mlb runs.
- `AgentRunServiceTests` (3 new): identity threaded on success; null passthrough; carried on the analyze-failure exception.
- `AgentRunsControllerTests` (6 new): columns persisted; OutputJson contains no identity properties; columns null without identity; failure path persists identity; artifact endpoint surfaces identity; legacy rows project null identity cleanly.
- `ToolGatewayMarketSpreadTests` updated to the grounding output types.

## Verification

- targeted suites green at each TDD step (red observed first in every phase: 12, 2, 2, 3 assertion failures respectively).
- full .NET suite: 634 passed, 0 failed. full solution build succeeded.
- frontend untouched and green: `git status apps/ services/` shows zero diffs; `ng test` 47 passed (5 files); `ng build` exit 0.
- `git diff --check` clean in both repos.
- OutputJson contract: explicit integration test proves no identity property appears in the serialized artifact.
- ascii: all hand-written added lines lowercase-ascii-comment clean; the only non-ascii in new files is the EF auto-generated designer reproducing the pre-existing seeded San Jose State team name, which contains an accented character (already present in prior migration designers, 2 occurrences each).
- migration not applied to the running dev db in this slice; it applies with `dotnet ef database update` (documented dev flow).

## Recommended next work

- MLB Game Identity Capture v1: capture `gamePk` + scheduled start from statsapi on the mlb path (provider `mlb_statsapi`), closing the buyer-ready-sport identity gap before more mlb season passes.
- Outcome Reconciliation Runtime v1 (remainder): the matcher joining settled games to runs on the canonical key, expanded settlement-status taxonomy, and a thin manual/import settlement path -- still provider-agnostic.
- begin manual Stage 0 reconciliation now on recently settled NBA/MLB runs via the existing `/outcome` endpoint to learn outcome-source reliability before automating.

## What was not changed

- no matcher, settlement provider, scheduled job, or outcome-status taxonomy change; `RunEvaluator` untouched
- no OutputJson artifact contract change; no buyer projection or buyer UI change; buyer-visible track record stays deferred
- no FastAPI prompt/parser change; no `SportsEvaluator`/`SportsComposer`/`SportsQualityChecker` behavior change
- no confidence/posture/lean/signal/model/cost change; no probe-refresh; no source expansion; no `CompetitionCatalog` readiness change
- no pricing/Stripe/auth/tenant/dashboard/deployment/Postgres change; no Jera change
