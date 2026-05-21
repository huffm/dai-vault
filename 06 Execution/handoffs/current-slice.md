# current slice

**slice:** cognitive protocol runtime — artifact contract migration (slice 4)
**status:** shipped 2026-05-14
**repos touched:** `dai` (.NET backend, tests); `dai-vault` (this doc, contract doc, sports cognitive worker model, sports flow)

## what shipped

The first runtime-side slice of the Cognitive Protocol Runtime migration. New sports runs persist a canonical `CognitiveProtocol` block alongside the legacy `CognitivePhases`. The persisted `AgentRunExecutionResult` now carries an explicit `ArtifactVersion`.

- canonical contract: `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocol.cs`
  - records: `CognitiveProtocol`, `PerceiveProtocol`, `InterrogateProtocol`, `DiscernProtocol`, `DecideProtocol`, `SynthesizeProtocol`
  - constants: `ArtifactVersions.SportsDecisionArtifactV2 = "sports_decision_artifact_v2"`
- deterministic builder: `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolBuilder.cs`
  - `CognitiveProtocolBuilder.FromLegacy(SportsCognitivePhases?)` produces the canonical shape from the legacy phases at compose time
- compose wiring: `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsComposer.cs`
  - success path stamps `ArtifactVersion` and populates `CognitiveProtocol`
  - failure path stamps `ArtifactVersion` and leaves `CognitiveProtocol` null
- read-side projection updated: `dai/platform/dotnet/DevCore.Api/AgentRuns/ProtocolVocabularyMapper.cs`
  - prefers persisted `CognitiveProtocol` when present (v2)
  - falls back to legacy `CognitivePhases` when absent (v1)
- DTO surface: `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs`
  - `AgentRunExecutionResult` gains optional `ArtifactVersion` and `CognitiveProtocol`
  - `AgentRunArtifactDto` surfaces both on `GET /api/agent-runs/{id}/artifact`
- view: `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolView.cs`
  - `DiscernStressProtocolView` gains optional `Canonical` slot for canonical-sourced stress
- tests: 24 new tests added (197 total passing)
  - `CognitiveProtocolBuilderTests.cs` covers the legacy → canonical builder rules
  - `ProtocolVocabularyMapperTests.cs` covers canonical-source preference, legacy fallback, and round-trip
  - `SportsComposerTests.cs` covers v2 stamping on success and failure paths and v1 deserialization
  - `AgentRunsControllerTests.cs` covers the artifact endpoint for both v1 and v2 records

## doctrine carried by this slice

- canonical protocol names are now authoritative on persisted artifacts. legacy names remain compatibility aliases only.
- Retrieve is platform plumbing, not a cognitive micro-action.
- `Interrogate.Probe` has no runtime source yet and is null on every v2 record. future slices may populate it.
- `Discern.Stress` is single-source by canonical contract. the builder prefers legacy `interrogate.stress` and falls back to legacy `discern.test` only when the first is null. no concatenation.
- `Decide.Position` preserves the validated posture string exactly. position is decision posture, not a pick.
- `Synthesize` is platform-operational, not simulated cognition.

## transitional compatibility model

The migration is dual-emit. While both shapes are present:

- new runs persist both `CognitivePhases` and `CognitiveProtocol`. They stay semantically aligned because they derive from the same single analyzer call.
- old runs (no `ArtifactVersion`, no `CognitiveProtocol`) continue to deserialize and project through the legacy fallback.
- the artifact inspection endpoint surfaces both shapes plus a `ProtocolView` projection. `ProtocolView` prefers canonical when present.
- the user-facing `AgentRunResultDto` is unchanged. Angular consumers are unaffected.
- there is no database migration, no schema change, no historical row rewrite, no calibration report rewrite.

## artifactVersion semantics

- `null` on `OutputJson` records: treat as v1. Legacy `CognitivePhases` only. Projection uses the legacy fallback path.
- `"sports_decision_artifact_v2"`: canonical `CognitiveProtocol` is present and authoritative. Legacy `CognitivePhases` is present for compatibility.
- the constant lives in `DevCore.Api.AgentRuns.ArtifactVersions.SportsDecisionArtifactV2`.

## what this slice did not touch

- `dai/services/agent-service/` (FastAPI Pydantic models, analyze prompt body, route handler)
- `DevCore.AiClient/SportAnalysisContracts.cs` (the wire contract for `SportsAnalysisResponse.Phases`)
- `SportsRunArtifact`, `SportsAnalyzer`, `SportsEvaluator`, `SportsQualityChecker` (and the pipeline stage order)
- Angular sports app or `/dev/artifacts` page
- database schema and migrations
- calibration reports under `dai-vault/04 Products/sports-v1/calibration/`

## future removal path for CognitivePhases

Removal lands in a later slice after the following:

1. Update the analyze prompt in `sports_analyzer.py` so the model emits canonical micro-action names directly.
2. Rename the FastAPI Pydantic models in `app/models/sports.py` and the .NET `SportsCognitivePhases` records in `DevCore.AiClient/SportAnalysisContracts.cs` in lockstep.
3. Adjust `SportsQualityChecker` rules that read legacy field-derived `CounterCase` / `WatchFor` to read canonical sources.
4. Confirm historical v1 records have either been migrated or aged out for the relevant calibration windows.
5. Drop `CognitivePhases` from `AgentRunExecutionResult` and the contract docs.

Until then, `CognitiveProtocol` is the authoritative source and `CognitivePhases` is a compatibility alias.

## next slice candidates

- Phase B continuation: dual-emit prompt update — teach the analyze prompt to emit canonical field names without removing legacy ones. Requires FastAPI + .NET lockstep.
- Probe runtime source: surface `SignalFollowUpRecord[]` into `Interrogate.Probe` so the canonical Probe is non-null when the platform has investigation candidates.
- Quality check migration: update `SportsQualityChecker` to read canonical sources for `CounterCase` / `WatchFor` derivation before the legacy fields are removed.

## addendum: Probe Population v1 (2026-05-14)

`Interrogate.Probe` is now populated deterministically at compose time from existing signal follow-up data. No LLM call. No FastAPI change. No prompt change. No database migration.

- new overload: `CognitiveProtocolBuilder.FromLegacy(SportsCognitivePhases?, SignalFollowUpRecord[]?)`. The single-arg overload (`FromLegacy(phases)`) is preserved and keeps Probe null.
- new helper: `CognitiveProtocolBuilder.BuildProbe(SignalFollowUpRecord[]?)` is the pure deterministic probe builder.
- `SportsComposer.Compose` calls the two-arg overload with `retrieval.SignalFollowUps`.
- failure path unchanged: `ComposeFailedRun` keeps `CognitiveProtocol` null, so Probe stays null on failure.

### gap detection

A follow-up record marks a missing primary when any of the following holds:

- `Reason == "primary_signal_missing"` -> `Signal` is the missing primary
- `DecisionUse == "missing_confirmation"` -> `Signal` is the missing primary
- `FallbackType == "lateral_proxy"` -> `TriggeredBy` is the missing primary (the proxy is filling for it)

Missing primaries are deduplicated and ordered by `StringComparer.Ordinal`.

### doctrinal probe templates

| missing signal | probe sentence |
|---|---|
| `sharp_public` | Sharp/public signal missing; market read relies on price movement only. |
| `market` | Market signal missing; directional read should rely on schedule and situational context. |
| `rest_schedule` | Rest edge is unclear; schedule signal should be treated as neutral. |
| `starting_pitching` | Starting pitching unavailable; lean should not depend on starter advantage. |

Signals without a template are silently dropped so the probe never fabricates injury, form, travel, or unsupported claims. `line_movement` is deliberately excluded because it is permanently `not_implemented` and would add noise on every run.

### dev artifact surface impact

`/dev/artifacts` already pass-through `cognitiveProtocol.interrogate.probe` from the artifact endpoint, so Probe now renders as a one or two sentence summary on v2 runs with real gaps and as "Not recorded" otherwise. No Angular change was needed.

## addendum: dev artifact surface (2026-05-14)

`/dev/artifacts` in `apps/sports-app` now renders the canonical Cognitive Protocol Runtime output in a new "Cognitive Protocol" section. The page:

- Shows an Artifact Version stat in the Run Overview. `v2 sports_decision_artifact` for new runs, `v1 (legacy)` for older runs.
- Prefers persisted `cognitiveProtocol` when present (v2 records).
- Falls back to the per-request `protocolView` projection when only the projection is available (v1 records).
- Displays "Not recorded" on every micro-action when both shapes are absent.
- Shows a source badge — "Canonical Artifact", "View Projection", or "Not Recorded" — so reviewers know which path produced the rendered block.
- Renders the five canonical cards: Perceive, Interrogate, Discern, Decide, Synthesize.
- For Discern.Stress: renders a single "Canonical Stress" row when canonical, or two labeled rows ("Legacy Interrogate Stress", "Legacy Discern Test") for v1 records. No concatenation.
- Leaves the legacy "Cognitive Phases" section in place as a labeled compatibility view.
- Does not change the customer-facing Matchup Analyzer output.

Backend, FastAPI, and database are unchanged in this surface slice. Only the dev review page and its TypeScript types were touched.

Affected files:

- `dai/apps/sports-app/src/app/core/models/agent-run.model.ts` — new TS types: `CognitiveProtocolDto`, `CognitiveProtocolViewDto`, plus nested per-protocol types; `AgentRunArtifactDto` gains optional `artifactVersion`, `cognitiveProtocol`, `protocolView`.
- `dai/apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.ts` — view-model builder (`protocolBlocks`), source resolution logic, multi-source Stress handling, artifact version label.
- `dai/apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.html` — Artifact Version stat in Run Overview; new Cognitive Protocol section with source badge.

## addendum: Outcome Reconciliation Harness v1 (2026-05-18)

Dev tooling to reconcile the 2026-05-18 cognitive protocol calibration batch against actual game outcomes once the games complete (~2026-05-20). No runtime code, no confidence rule, no analyzer or FastAPI change. It consumes only existing endpoints.

- new script: `dai/scripts/dev/sports/reconcile-calibration-outcomes.ps1`
  - reads the committed export csv plus a game-results input csv, matches by `run_id`
  - for each game marked `final` it posts to `POST /api/agent-runs/{id}/outcome`, then reads back `GET /api/agent-runs/{id}/evaluation` (the platform's directional eval)
  - writes a reconciled csv (only the 7 outcome columns filled) and a markdown reconciliation note; never overwrites the original export
  - `-DryRun` posts nothing and requires no running stack
  - cover / total result are filled only when `closing_spread_home` / `closing_total` are supplied in the results input; left blank otherwise — never guessed
- new template: `dai-vault/04 Products/sports-v1/calibration/protocol-runs/2026-05-18-game-results-input.csv`
  - pre-seeded with the 8 batch run ids, all `game_status = pending`
- outputs (same protocol-runs folder):
  - `2026-05-18-cognitive-protocol-run-export-reconciled.csv`
  - `2026-05-18-cognitive-protocol-outcome-reconciliation.md`

status: harness built and dry-run verified 2026-05-18 (8 runs, 0 completed, 8 pending; final-game derivation self-tested with synthetic scores). live reconciliation runs after 2026-05-20 once NBA (05-18, 05-19) and MLB (05-18) games are final and the results input is filled with actual scores. confidence calibration remains blocked on this reconciliation evidence — see the calibration export note.

## addendum: Cognitive Protocol Node Specs v1 (2026-05-18)

Doctrine slice. Vault docs only — no runtime code touched.

- new doc: `dai-vault/02 Platform/architecture/cognitive-factory/protocol-node-specs.md`
  - operational node spec for all 15 cognitive nodes (Perceive/Interrogate/Discern/Decide + Synthesize, three micro-actions each)
  - each node defines 11 facets: purpose, fields read, fields written, allowed tools, allowed scripts/reflexes, model-call rule, signal dependencies, quality checks, fallback, sports overlay, forbidden behavior
  - sits below `cognitive-protocol-runtime.md` (names the nodes) and `protocol-vocabulary-map.md` (legacy↔canonical) as the contract a future code slice implements against
- doctrine reinforced by this doc:
  - Retrieve is platform plumbing, not a node; Perceive consumes retrieval output
  - Probe is deterministic (`CognitiveProtocolBuilder.BuildProbe` from `SignalFollowUpRecord[]`), no model call
  - the Synthesize trio is deterministic (`SportsComposer`), no model call
  - models never choose their own tools, write scope, or data access — fields and tools are fixed per node
  - confidence value and band are deterministic (`SportsEvaluator`); the model owns rationale text, not numbers or the posture enum
- open question carried forward: NCAAW (women's college basketball) is not a platform competition — no `CompetitionCatalog` entry, no teams, no retriever — so no node overlay exists for it. Decide whether to add the competition or drop it from cognitive scope.

status: node specs defined 2026-05-18. they give the stable target that the Phase B canonical prompt/contract rename was waiting on.

## addendum: Cloud and Tool Runtime Plan (2026-05-20)

Doctrine slice. Vault docs only. No runtime code, no FastAPI change, no database change, no Angular change.

- new doc: `dai-vault/02 Platform/architecture/cloud-tool-runtime-plan.md`
  - launch-friendly cloud target: Azure Container Apps for the .NET orchestrator and the FastAPI analyzer; Azure SQL as system of record; Static Web Apps for the Angular client; Key Vault for secrets; Application Insights for telemetry; Entra External ID for customer identity
  - explicit launch exclusions: AKS, multi-region, brokers, API Management, pgvector as primary store, MCP on the synchronous run path
  - DAI Tool Gateway named as the authority over every tool call, retriever call, FastAPI call, future MCP call, and future Azure Function invocation. Lives in `DevCore.Api` and wraps existing typed HttpClients first, then accumulates.
  - Tool Registry v1 shape: static manifest in code with `tool_id`, `kind`, `transport`, `handler`, `allowed_protocol_nodes`, `secrets_scope`, `idempotency`, `cache_ttl`, `cost_class`, `tenant_tier_minimum`, `calibration_hooks`. Designed so a database-backed dynamic registry can be added later without rewriting the gateway interface.
  - Azure Functions positioned for bounded reflex jobs and scheduled or event-driven tools only, never on the synchronous run path
  - pgvector positioned as additive memory and search, not a primary store replacement, with `allowed_protocol_nodes` initially empty for synchronous use and first realistic opening at Interrogate.Probe
  - MCP positioned as a future transport value in the registry, not the core runtime; the gateway hides the transport from protocol nodes
  - AKS positioned as future infrastructure, only when Container Apps proves insufficient
- doctrine reinforced by this doc:
  - the runtime tells you what cognition is allowed; this doc tells you what infrastructure cognition runs on and what governs the tools cognition is allowed to invoke
  - .NET keeps orchestration, retrieval, evaluation, synthesis, identity, and now the Tool Gateway; FastAPI stays narrow on the analyze model call only
  - models never choose tools; the gateway, not the model, decides what a node may invoke (consistent with `protocol-node-specs.md`)
- recommended next slices (sequenced):
  1. tool gateway skeleton in .NET wrapping `OddsScheduleClient.GetEventsAsync` only
  2. wrap remaining typed clients (`OddsMarketClient`, `EspnBasketballScheduleClient`, `MlbStarterClient`) and the FastAPI analyze call behind the gateway
  3. container apps deploy slice: package both services, internal DNS, Key Vault, App Insights, smoke parity with `test-sports-dev.ps1`
  4. pgvector memory landing for calibration corpora, offline first, no synchronous run dependency
  5. first azure function tool: outcome reconciliation timer, re-hosting `reconcile-calibration-outcomes.ps1`
  6. probe enrichment via gateway, behind a per-tenant flag, as the first real opening of `allowed_protocol_nodes` for a non-typed-client tool
- open questions carried forward:
  - does payment-tier enforcement on tool calls land before or after the first paying customer? Registry has the field; enforcement turns on after Stripe webhook truth is wired through.
  - which competition gets the first pgvector-backed Probe enrichment? candidates are NBA (mature signal set) or MLB (single-signal runs benefit most from prior-run lookups).
  - region selection at launch.

status: plan written 2026-05-20. it gives the stable launch target the Phase B prompt/contract rename and the first Tool Gateway slice can be planned against without ambiguity.

## addendum: Tool Gateway Wire-In v1 (2026-05-20)

Runtime slice. Wires the skeleton into DI and migrates one safe call site. No FastAPI change, no DB schema change, no customer-facing output change.

- new code (`dai/platform/dotnet/DevCore.Api/Tools/`):
  - `ToolGatewayServiceCollectionExtensions.cs` — `AddDaiToolGateway(IServiceCollection)` extension. Registers `IToolRegistry` (singleton, `ToolRegistry.Default()`), `IToolGateway` (scoped), and `IToolHandler<ScheduleMatchupDatesInput, MatchupEventDto[]>` as `AddKeyedScoped` keyed by `ToolIds.ScheduleMatchupDates`. Scoped lifetimes match `AddHttpClient<OddsScheduleClient>`'s transient-with-managed-handler so the gateway can resolve handlers per request without holding HTTP message handlers alive.
- modified code:
  - `dai/platform/dotnet/DevCore.Api/Program.cs` — adds `using DevCore.Api.Tools;` and one line: `builder.Services.AddDaiToolGateway();` after the existing pipeline registrations.
  - `dai/platform/dotnet/DevCore.Api/Controllers/SportsReferenceController.cs` — constructor now also injects `IToolGateway` alongside the existing `OddsScheduleClient` and `AppDbContext`. The `GetMatchupDates` action invokes the gateway with `ProtocolNode = ProtocolNodes.PlatformReference`, `CorrelationId = HttpContext.TraceIdentifier`, `RunId = null`, `TenantKey = null`. `GetUpcoming` is unchanged — it still calls `OddsScheduleClient.GetAllUpcomingEventsAsync` directly because that method is not wrapped by the v1 registry.
- new tests (`dai/platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs`):
  - `resolve_tool_gateway_from_application_services` — resolves `IToolGateway` from the real `Program.cs` service graph and asserts it is a `ToolGateway`.
  - `resolve_tool_registry_with_default_manifest_containing_schedule_matchup_dates` — asserts the v1 manifest entry is present with the expected kind, transport, and allowed caller.
  - `resolve_schedule_matchup_dates_handler_as_keyed_service` — asserts the keyed handler resolves to `ScheduleMatchupDatesHandler`.
- new tests (`dai/platform/dotnet/DevCore.Api.Tests/Integration/SportsReferenceControllerTests.cs`):
  - `matchup_dates_returns_not_found_for_unknown_competition_code` — 404 path still fires before the gateway is reached.
  - `matchup_dates_returns_bad_request_when_team_is_missing` — 400 path still fires before the gateway is reached.
  - `matchup_dates_returns_ok_with_empty_array_via_tool_gateway_when_provider_unavailable` — the gateway-routed happy path returns the same JSON array shape; team seeding added to `InitializeAsync` so the controller's team-existence check passes.
- behavior compatibility: the API response for `GET /api/competitions/{code}/matchup-dates` is byte-identical at the JSON-array level. Same content type, same status codes for the unchanged validation branches, same empty-on-degraded behavior when `OddsApi:ApiKey` is unset. Existing `SportsReferenceController` callers, the Angular sports app, and `/dev/artifacts` consumers see no contract change.
- what stayed deferred: correlation header injection into outbound HTTP from the gateway, idempotency caching, cost-class enforcement, tenant-tier enforcement, swapping the retrieval-side call sites (`OddsMarketClient`, `EspnBasketballScheduleClient`, `MlbStarterClient`, `FastApiClient`), wiring into the main sports analysis pipeline, Azure Container Apps deploy, pgvector landing, Azure Functions, MCP adapters.
- next safe slice: wrap the next typed client behind the gateway. Recommended order: `OddsMarketClient.GetFootballSpreadAsync` and `GetBasketballSpreadAsync` (retrieve-stage callers, `ProtocolNodes.PlatformRetrieve`). That slice opens `platform.retrieve` to the manifest, exercises the same wrap-in-handler pattern, and stays inside `dai/platform/dotnet/` with no FastAPI or DB changes.

status: wire-in merged 2026-05-20. tests: 216 passed, 0 failed (was 210, +6).

## addendum: Cognitive Protocol Quality Surface Audit v1 (2026-05-20)

Audit slice triggered by Outcome Reconciliation v1: several MLB runs had `evidence_richness=1` and `confidence >= 0.70` but the export's `quality_warnings` showed `none`, and one (Braves at Marlins) was a 12-0 directional miss. The question was whether `confidence_high_for_partial_evidence` is computed, stored, surfaced, and exported correctly.

### finding (this was an export gap, not a runtime gap or docs mismatch)

- `confidence_high_for_partial_evidence` **is** computed, by the offline harness `dai/scripts/dev/sports/run-artifact-calibration.ps1:435-439` (rule: `evidence_richness < 3 AND confidence >= 0.70`). It is **not** computed in .NET runtime code and is **not** stored on the artifact.
- The flag fired correctly on all the runs it should have: the 2026-05-18 harness reports (`20260518-1001-nba-calibration.md`, `20260518-1010-mlb-calibration.md`) show it on both NBA runs and 3 of 6 MLB runs (5 of 8 total). The offline computation was never broken.
- The `quality_warnings` column in the cognitive protocol export maps to `ArtifactQualityWarnings` (the runtime `SportsQualityChecker` output). That field carries only signal-narrative-drift and `signals_used` integrity warnings. **Calibration flags and `ArtifactQualityWarnings` are separate concepts** and were never the same surface.
- Root cause: the cognitive protocol run export (hand-curated to `2026-05-18-cognitive-protocol-run-export.csv`) and the reconciliation export did not carry calibration flags at all, so a reviewer scanning `quality_warnings` saw `none` and concluded the risk was invisible. The risk was visible — in a different file (the harness markdown) the reviewer was not looking at.

classification: **export gap.** Not a runtime gap (the offline flag works), not a docs mismatch (`protocol-node-specs.md` correctly calls it a "calibration flag"), not a naming mismatch (the names are consistent; the two surfaces are just genuinely distinct).

### fix (smallest, additive, no runtime change)

- `dai/scripts/dev/sports/reconcile-calibration-outcomes.ps1`:
  - new helper `Get-DerivedCalibrationFlags` derives `confidence_high_for_partial_evidence` from each export row's `confidence` / `evidence_richness`, with the rule cited to `protocol-node-specs.md` and parity to `run-artifact-calibration.ps1:435-439`.
  - new reconciled-output column `derived_calibration_flags` (additive; the original export CSV is untouched).
  - new "## derived calibration flags" section in the reconciliation markdown note, with a count table and per-run breakdown, plus an explicit caveat that this is distinct from `quality_warnings`.
- regenerated `2026-05-18-cognitive-protocol-run-export-reconciled.csv` and `...-outcome-reconciliation.md` (live re-run; outcomes already recorded so the harness read evaluations via 409, directional eval preserved). 5 of 8 runs now visibly carry `confidence_high_for_partial_evidence` next to their directional eval: Spurs/Thunder (incorrect), Cavs/Knicks (correct), Braves/Marlins (incorrect), Guardians/Tigers (correct), Blue Jays/Yankees (correct).
- `protocol-node-specs.md` global rules: added one rule distinguishing calibration flags from `ArtifactQualityWarnings` so the next reader does not repeat the confusion.

what did NOT change: no confidence values, no confidence thresholds, no model prompts, no FastAPI, no Tool Gateway behavior, no legacy `CognitivePhases` removal, no .NET runtime code, no DB migration. The warning only makes the risk visible; it does not change lean, confidence, or position. Verified by dry-run (scratch dir) and the regenerated live artifacts.

### the confidence question, with evidence

The 5/8 flag rate plus the directional results (the two `incorrect` leans, Spurs/Thunder and Braves/Marlins, both carried the flag) is suggestive but not actionable: 5 leans is too small to change a threshold. The flag is now visible on the reconciled export so the next calibration window can accumulate evidence. Confidence Calibration Rules v1 stays deferred until there is enough.

### Tool Gateway continuity

- Tool Gateway Wire-In v1 is complete (`SportsReferenceController.GetMatchupDates` routes through `IToolGateway`; dai @ `6505fb6`).
- The next Tool Gateway slice remains unchanged: wrap `OddsMarketClient.GetFootballSpreadAsync` and `GetBasketballSpreadAsync` behind the gateway, opening `ProtocolNodes.PlatformRetrieve` in the manifest.
- This quality surface audit was done first because Outcome Reconciliation v1 exposed an artifact quality visibility gap that was cheap to close and blocked clear reading of calibration evidence. With the gap closed, Tool Gateway expansion is the next slice.

status: audit complete 2026-05-21. export fix verified by PowerShell dry-run + regenerated live artifacts. no .NET code changed, so no dotnet test delta (suite remains 216 passing from the wire-in slice).

### local skills used (Claude Code, jera-workspace-skills pack)

- `dai-grill-with-vault` — applied its read-before-conclude discipline: grepped the literal flag across both repos, traced `quality_warnings` to `ArtifactQualityWarnings`, and located the rule in `run-artifact-calibration.ps1` and `protocol-node-specs.md` before deciding the fix. Used the reading half, not the interactive-grill half (no user Q&A — this was a solo diagnostic, not a fuzzy plan).
- `dai-token-tight` — reporting density.
- `superpowers:verification-before-completion` — every claim backed by grep/dry-run/regenerated artifact; ASCII + AST parse-check run explicitly before close.

skill-fit note (for later, do not action now): `dai-grill-with-vault` is shaped for interactive plan interrogation — its closing template (locked decisions, deferred decisions with owner, recommended next prompt) does not map cleanly onto a solo code/vault audit that ends in a fix. A dedicated `dai-audit` or `dai-vault-diagnose` skill (read repo+vault, classify gap type, smallest fix, evidence trail) would fit this slice's shape better. Recommend sharpening later; not changed in this slice (jera-workspace-skills left untouched, no approval to edit).

### Claude <-> Codex transfer notes

- Repos in play this slice: `dai` (1 file: the reconcile script) and `dai-vault` (4 files: protocol-node-specs rule, regenerated reconciled CSV + MD, this handoff). `jera-workspace-skills` read-only, untouched.
- The fix is PowerShell + docs only. No build step. To re-verify on any machine: ASCII scan + `[System.Management.Automation.Language.Parser]::ParseFile` (both clean), then `reconcile-calibration-outcomes.ps1 -DryRun -OutputDir <scratch>` and confirm `derived_calibration_flags` fires on the 5 runs with confidence >= 0.70 and richness < 3.
- Live regeneration requires the stack: Docker Desktop + `devcore-sql` container (it OOM-exited 137 overnight twice this slice — restart with `docker start devcore-sql`, wait for "Recovery is complete"), then the .NET API on :5007. The 8 run ids already have outcomes recorded, so a live re-run returns 409 and reads evaluations idempotently — safe to re-run.
- Open thread for the next agent: confidence calibration is unresolved by design (5 leans is too small). The flag is now visible on the reconciled export; accumulate more game-day reconciliations before touching thresholds.
- Next slice is Tool Gateway expansion (`market.football.spread`, `market.basketball.spread`), not a confidence change.
