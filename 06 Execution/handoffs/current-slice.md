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

## addendum: Tool Gateway Market Spread Wrap v1 (2026-05-21)

Runtime slice. Routes market spread retrieval through the Tool Gateway. No FastAPI, no prompt, no confidence-rule, no CognitiveProtocolBuilder, no DB, no Angular, no MCP, no Azure, no pgvector changes.

### local skills used (Claude Code)

- `dai-grill-with-vault` (jera pack) — ran the naming gate against doctrine before code: confirmed `market.football.spread` / `market.basketball.spread` against `cloud-tool-runtime-plan.md` section 5 (where they are the literal examples) and the `schedule.matchup_dates` dotted-namespace precedent.
- `dai-token-tight` (jera pack) — reporting density.
- `superpowers:test-driven-development` — RED (new dispatch tests fail to compile) -> GREEN (registry + handlers + DI) verified.
- `superpowers:verification-before-completion` — full suite run before any completion claim; new tests confirmed by name.
- `dai-agent-handoff` shaping for the transfer notes below.
- `dai-signal-follow-up-diagnostics` considered and skipped — it diagnoses signal coverage gaps on a run, not gateway plumbing.

skill-fit note (carryover, not actioned): `dai-grill-with-vault` again did the read+verify job but its interactive closing template still does not match a solo implementation slice. The earlier recommendation for a dedicated `dai-audit` / `dai-implement-with-vault` skill stands. jera-workspace-skills left untouched this slice (no approval to edit).

### naming review result

Confirmed, no rename. `market.football.spread` / `market.basketball.spread` are: specific (domain.sport.capability), stable, boring, doctrine-aligned (literal examples in the cloud plan), consistent with the dotted-namespace precedent of `schedule.matchup_dates`, and not misleading (they name retrieval, not cognition). Handler classes follow the tool-id-to-PascalCase convention (`MarketFootballSpreadHandler`, `MarketBasketballSpreadHandler`). One deliberate decision: a single shared `MarketSpreadInput(Competition, HomeTeam, AwayTeam, GameDate)` record rather than two identical per-tool records — the input shape is identical; the tools differ by output context type (`FootballMarketContext?` vs `BasketballMarketContext?`) and underlying client method. Allowed node is the existing `ProtocolNodes.PlatformRetrieve` (no new node).

### files changed

- `dai/platform/dotnet/DevCore.Api/Tools/ToolRegistry.cs` — `ToolIds.MarketFootballSpread`, `ToolIds.MarketBasketballSpread`; two `ToolDefinition` entries (Retriever, HttpExternal, `platform.retrieve`, PerRunInput, 15-min cache mirroring `OddsMarketClient`, PaidExternal).
- `dai/platform/dotnet/DevCore.Api/Tools/Handlers/MarketSpreadHandlers.cs` (new) — `MarketSpreadInput` + `MarketFootballSpreadHandler` + `MarketBasketballSpreadHandler` wrapping `OddsMarketClient.GetFootballSpreadAsync` / `GetBasketballSpreadAsync` 1:1.
- `dai/platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` — two keyed-scoped handler registrations (+ `using DevCore.AiClient;`).
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsRetriever.cs` — constructor now takes `IToolGateway` instead of `OddsMarketClient`; football and basketball spread calls route through `gateway.InvokeAsync(...)` with a `platform.retrieve` context carrying `RunId = artifact.AgentRunId`. All other retrieval (starters, basketball schedule/rest, sharp/public) unchanged.
- `dai/platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayMarketSpreadTests.cs` (new) — 3 tests: football dispatch, basketball dispatch, denied-node.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsRetrieverTests.cs` — `MakeRetriever` builds a real gateway wired to the fake-http `OddsMarketClient`; new `retrieve_grounds_basketball_market_through_gateway_for_nba` test.
- Program.cs unchanged: `AddDaiToolGateway()` already registers the gateway and now the two new handlers; `OddsMarketClient` stays registered (the handlers depend on it).

### behavior summary

API and artifact behavior are identical. Market spread for nfl/ncaaf and nba/ncaamb now flows `SportsRetriever -> IToolGateway -> Market*SpreadHandler -> OddsMarketClient`, instead of a direct client call. Null-on-failure, missing-signal computation, degradation notes, `SportsRetrievalOutput` shape, and grounded-signal sets are unchanged (proven by the pre-existing nfl retriever tests plus the new nba test, all green). The gateway enforces `platform.retrieve` and fails closed for any other caller. `SportsRetriever` no longer references `OddsMarketClient` directly.

### test results

`dotnet test`: 220 passed, 0 failed (was 216, +4). New: `invoke_dispatches_market_football_spread_to_odds_market_client_when_node_is_allowed`, `invoke_dispatches_market_basketball_spread_to_odds_market_client_when_node_is_allowed`, `invoke_throws_tool_not_allowed_when_market_football_spread_called_from_non_retrieve_node`, `retrieve_grounds_basketball_market_through_gateway_for_nba`. Existing nfl retriever tests pass unchanged (regression guard for the gateway routing).

### risks

- `SportsRetriever` is now scoped-resolved with a scoped `IToolGateway` and scoped keyed handlers; validated end to end by the WebApplicationFactory integration tests in the suite. No lifetime mismatch observed.
- Correlation header injection still not implemented at the gateway (deferred). `RunId` is now carried on the retrieve context but is not yet emitted as `X-Agent-Run-Id` on the outbound odds-api call.
- Idempotency/cost-class metadata on the new tool entries is declarative only; not enforced yet (same as `schedule.matchup_dates`).
- `OddsMarketClient`'s own 15-min `MemoryCache` is still the live cache; the registry `CacheTtl` is documentation, not a second cache.

### next recommended slice

Wrap the remaining retrieve-stage clients behind the gateway for parity: `EspnBasketballScheduleClient.GetRestContextAsync` (`schedule.basketball.rest`), `MlbStarterClient.GetStartersAsync` (`starters.mlb`), and `ActionNetworkClient.GetSharpPublicDataAsync` (`market.sharp_public` or `signal.sharp_public`). After all retrieve-stage tools are gateway-routed, do the gateway correlation-header + telemetry slice (emit `X-Agent-Run-Id` / cost-class counters), then the analyze-call wrap (`analyze.sports`). Container Apps deploy comes after the retrieve-side is fully behind the gateway.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (6 files: registry, handlers, extensions, retriever, 2 test files) and `dai-vault` (this handoff). `jera-workspace-skills` read-only, untouched.
- Re-verify on any machine: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> expect 220 passing. No stack, no DB, no network needed (all tests use fake HTTP handlers).
- The pattern to extend: add `ToolDefinition` + `ToolIds` constant in `ToolRegistry.cs`, implement `IToolHandler<TInput, TOutput>` in `Tools/Handlers/`, register keyed-scoped in `ToolGatewayServiceCollectionExtensions.cs`, then route the call site through `gateway.InvokeAsync(...)` with the correct `ProtocolNodes.*`. Nullable reference outputs (`FootballMarketContext?`) are fine as the generic `TOutput` — the annotation is erased so registration and resolution match.
- No PowerShell changed this slice, so no ASCII/parser-validation step was required.
- Open thread unchanged: confidence calibration still needs more reconciled game-days before any threshold change.

status: market spread wrap merged 2026-05-21. tests 220 passing (was 216, +4). no FastAPI/prompt/confidence/DB/Angular/MCP/Azure/pgvector changes. jera-workspace-skills untouched.

## addendum: Tool Gateway Retrieve Parity v1 (2026-05-21)

Runtime slice. Routes the remaining three retrieve-stage signals through the Tool Gateway. After this slice every external retrieval call in `SportsRetriever` goes through `IToolGateway`. No FastAPI, prompt, confidence-rule, CognitiveProtocolBuilder, DB, Angular, MCP, Azure Functions, or pgvector changes.

### naming and skills gate

A naming gate ran before code (preferred names were challenged against the real DTOs and rejected where misleading). Final tool ids, approved:
- `schedule.basketball.rest_context` (not `rest_edges`: the DTO returns raw rest facts, not a computed edge)
- `pitching.mlb.probable_starters` (durable `pitching` domain; `probable_starters` is the real MLB concept; no starters/pitchers stutter)
- `market.sharp_public.split` (domain.signal.capability; `split` names the bet%/money% payload; cross-sport, so the signal sits in the middle slot deliberately, not a sport family)

Skills used: `dai-grill-with-vault` (read clients/DTOs/`SportsRetriever` before naming), `dai-token-tight` (reporting), `superpowers:test-driven-development` (RED 68 errors -> GREEN), `superpowers:verification-before-completion` (named-test confirmation + full suite), `dai-agent-handoff` shaping. Skill-fit note carried forward unchanged: a dedicated `dai-audit`/`dai-implement-with-vault` skill would fit solo implementation slices better than the interactive grill template. jera-workspace-skills untouched (no approval to edit).

### files changed

- `dai/platform/dotnet/DevCore.Api/Tools/ToolRegistry.cs` — three `ToolIds` constants + three `ToolDefinition` entries (Retriever, HttpExternal, `platform.retrieve`, PerRunInput; cache ttls mirror clients: rest 30m, starters 30m, sharp_public 15m; CostClass Free because all three providers are keyless public endpoints).
- `dai/platform/dotnet/DevCore.Api/Tools/Handlers/RetrieveSignalHandlers.cs` (new) — `MatchupRetrievalInput` (shared by rest_context + sharp_public), `MlbProbableStartersInput` (no competition field), and the three handlers wrapping `EspnBasketballScheduleClient.GetRestContextAsync`, `MlbStarterClient.GetStartersAsync`, `ActionNetworkClient.GetSharpPublicDataAsync` 1:1. The sharp_public handler returns the full `SharpPublicLookupResult` (status + context), not a bare nullable.
- `dai/platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` — three keyed-scoped handler registrations.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsRetriever.cs` — constructor reduced to `(IToolGateway)`. The three remaining direct client calls (rest, starters, both sharp_public branches) now route through `gateway.InvokeAsync(...)` with the `platform.retrieve` context carrying `RunId = artifact.AgentRunId`. `SportsRetriever` no longer references `MlbStarterClient`, `EspnBasketballScheduleClient`, or `ActionNetworkClient` directly.
- `dai/platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayRetrieveParityTests.cs` (new) — registry assertions, keyed-handler resolution, sharp_public grounded dispatch, mlb starters grounded dispatch, rest routing, denied-node.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsRetrieverTests.cs` — `MakeRetriever` builds a gateway wired to all five handlers and constructs `SportsRetriever(gateway)`; optional `mlbJson` param; new `retrieve_grounds_starting_pitching_through_gateway_for_mlb` integration test.
- Program.cs unchanged: `AddDaiToolGateway()` already registers gateway + handlers; the three clients stay registered via `AddHttpClient<T>` (now consumed by handlers, not the retriever).

### behavior summary

API and artifact behavior identical. All retrieve-stage signals (football market, basketball market, basketball rest, mlb starters, sharp/public for football and basketball) now flow `SportsRetriever -> IToolGateway -> handler -> typed client`. Null-on-failure, `SharpPublicLookupResult` status semantics, missing-signal computation, degradation notes, `SportsRetrievalOutput` shape, and grounded-signal sets are unchanged. Proven by the pre-existing nfl tests (sharp_public + football market grounded through the gateway), the nba market test, and the new mlb starters test. Gateway enforces `platform.retrieve` and fails closed for any other caller.

### test results

`dotnet test`: 227 passed, 0 failed (was 220, +7). New: 6 in `ToolGatewayRetrieveParityTests` (registry, keyed resolution, sharp_public dispatch, mlb starters dispatch, rest routing, denied-node) + `retrieve_grounds_starting_pitching_through_gateway_for_mlb`. Pre-existing `xUnit2013` warning in `AgentRunsControllerTests.cs:583` is unrelated.

### risks

- `SportsRetriever` now depends solely on the gateway; if a tool id or handler registration is dropped from `AddDaiToolGateway`, retrieval fails closed at runtime. Covered by the registry + DI-resolution tests and the WebApplicationFactory integration tests in the suite.
- Correlation-header injection still deferred: `RunId` is carried on the context but not yet emitted as `X-Agent-Run-Id` on outbound calls.
- `MatchupRetrievalInput` and the committed `MarketSpreadInput` are shape-identical; converging them is a future cleanup, intentionally not done here.
- Cache/idempotency/cost-class metadata on the new entries is declarative; each client's own `MemoryCache` is still the live cache.

### next slice

The retrieve stage is fully behind the gateway. Two reasonable next moves: (a) the gateway correlation-header + telemetry slice (emit `X-Agent-Run-Id`, per-tool duration/outcome/cost-class counters), or (b) wrap the analyze model call as `analyze.sports` (allowed node = the analyze stage). Recommend (a) first: it is small, touches only the gateway, and makes every already-wrapped tool observable before the analyze wrap adds the model-call surface. Container Apps deploy comes after.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (6 files) and `dai-vault` (this handoff). `jera-workspace-skills` read-only, untouched.
- Re-verify anywhere: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 227 passing. No stack/DB/network needed (fake HTTP handlers throughout).
- Extension pattern unchanged from the market slice: `ToolDefinition` + `ToolIds` constant, `IToolHandler<TInput,TOutput>` in `Tools/Handlers/`, keyed-scoped registration, then route the call site through `gateway.InvokeAsync(...)` with `ProtocolNodes.PlatformRetrieve`. Handlers may return a non-nullable wrapper (`SharpPublicLookupResult`) or a nullable context; both work as `TOutput`.
- No PowerShell changed this slice, so no ASCII/parser-validation step was required.

status: retrieve parity merged 2026-05-21. tests 227 passing (was 220, +7). every retrieve-stage signal now routes through the Tool Gateway. no FastAPI/prompt/confidence/CognitiveProtocolBuilder/DB/Angular/MCP/Azure/pgvector changes. jera-workspace-skills untouched.

## addendum: Tool Gateway Correlation and Telemetry v1 (2026-05-21)

Runtime slice. Makes every gateway invocation observable via structured logging, without changing behavior. No FastAPI, prompt, confidence-rule, CognitiveProtocolBuilder, DB, Angular, MCP, Azure Functions, pgvector, or analyze.sports changes.

### naming and skills gate

Skills: `dai-grill-with-vault` (read the gateway + existing client logging convention + cloud-plan telemetry list before deciding names), `dai-token-tight` (reporting), `superpowers:test-driven-development` (RED 10 errors -> GREEN), `superpowers:verification-before-completion` (fresh suite + named-test confirmation), `dai-agent-handoff` shaping. Skill-fit note carried forward: a dedicated `dai-audit`/`dai-implement-with-vault` skill would fit solo implementation slices better than the interactive grill template. jera-workspace-skills untouched (no approval to edit).

Naming decisions (no competing user-preferred names, so adopted directly):
- telemetry event: `ToolGatewayInvocation`, `EventId(1001)` (`ToolTelemetryEvents.Invocation`). One event per invocation.
- outcome vocabulary (helper enum `ToolInvocationOutcome`): `Success`, `Denied`, `NotRegistered`, `HandlerError` -- matches the scope's required outcome set.
- structured property tokens (PascalCase, idiomatic .NET; map 1:1 to the doctrine snake_case fields in cloud-tool-runtime-plan.md section 4): `ToolId`, `ProtocolNode`, `RunId`, `TenantKey`, `CorrelationId`, `CostClass`, `Outcome`, `DurationMs`.
- log scope: a correlation dictionary (`ToolId`, `ProtocolNode`, `RunId`, `TenantKey`, `CorrelationId`) wrapping the handler call so handler/client logs inherit the correlation.
- header names: `X-Agent-Run-Id` (canonical) and `X-Correlation-Id` stay the doctrine names; outbound injection is DEFERRED (see risks).
- metric counters: DEFERRED. Structured logging only this slice (App Insights surfaces it as traces with custom dimensions); System.Diagnostics.Metrics counters would be a heavier abstraction than warranted now.

### files changed

- `dai/platform/dotnet/DevCore.Api/Tools/ToolTelemetry.cs` (new) -- `ToolInvocationOutcome` enum and `ToolTelemetryEvents.Invocation` event id.
- `dai/platform/dotnet/DevCore.Api/Tools/ToolGateway.cs` -- constructor gains `ILogger<ToolGateway>`. Each invocation opens a correlation log scope, times with `Stopwatch.GetTimestamp/GetElapsedTime`, and emits one `ToolGatewayInvocation` entry per outcome: Success=Information, Denied=Warning, NotRegistered/HandlerError=Error. Fail-closed paths still throw; handler exceptions are logged and re-thrown unchanged (telemetry never swallows).
- `dai/platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayTelemetryTests.cs` (new) -- recording logger + isolated single-tool registry + spy handler; 4 tests (success metadata, denied + no-invoke, not_registered, handler_error + rethrow).
- `dai/platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayTests.cs`, `ToolGatewayMarketSpreadTests.cs`, `ToolGatewayRetrieveParityTests.cs`, `DevCore.Api.Tests/AgentRuns/SportsRetrieverTests.cs` -- added `services.AddLogging()` to the hand-built providers so the gateway's new `ILogger` dependency resolves.
- Program.cs unchanged: the default host already registers logging, so DI provides `ILogger<ToolGateway>` automatically.

### behavior summary

No behavior change. Every `ToolGateway.InvokeAsync` now emits exactly one structured log entry with tool id, protocol node, outcome, cost class, duration_ms, and run/tenant/correlation ids when present. Denied and not_registered remain fail-closed (handler never resolved or invoked). Handler exceptions are logged at Error and re-thrown as the same instance -- the analyze-failure / ComposeFailedRun path and all existing exception semantics are preserved. Handler outputs are untouched.

### test results

`dotnet test`: 231 passed, 0 failed (was 227, +4). New: `successful_invocation_logs_success_outcome_with_metadata`, `denied_node_logs_denied_outcome_and_never_invokes_handler`, `unknown_tool_logs_not_registered_outcome`, `handler_error_logs_handler_error_outcome_and_rethrows`. WebApplicationFactory integration tests pass, confirming the 3-arg gateway resolves through the real Program.cs DI graph.

### risks

- Outbound correlation-header injection is DEFERRED, by design. The wrapped tools are external keyless providers (odds api, espn, statsapi.mlb.com, actionnetwork) that ignore `X-Agent-Run-Id`, and injecting our run id into third-party calls has no value and mild leakage risk. The header that matters is the internal `.NET -> FastAPI` call, which belongs with the analyze.sports wrap (explicitly out of scope here). Documented, not done.
- Telemetry is structured-logging-only; there are no queryable metric counters yet. Adequate for App Insights trace dimensions; revisit if dashboards need pre-aggregated counters.
- `HandlerError` is logged for any non-cancellation and cancellation exception alike (catch-all), then re-thrown. Behavior is unchanged; only the log classification is coarse.
- Structured property tokens are PascalCase per .NET convention, not the doctrine snake_case; the mapping is 1:1 and documented here so log queries are predictable.

### next slice

Wrap the analyze model call as `analyze.sports` through the gateway (allowed node = the analyze stage). That slice is where outbound correlation-header injection (`X-Agent-Run-Id` / `X-Correlation-Id` on the internal FastAPI call) becomes meaningful and should land together with the wrap. After that, the Container Apps deploy slice. Metric counters remain optional and demand-driven.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (6 files: 2 new, 4 modified) and `dai-vault` (this handoff). `jera-workspace-skills` read-only, untouched.
- Re-verify anywhere: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 231 passing. No stack/DB/network needed.
- The telemetry seam is `ToolGateway.LogOutcome`; any new outcome dimension is added there plus the `ToolInvocationOutcome` enum. New tools need nothing telemetry-side -- they inherit the invocation log automatically.
- Gotcha for future gateway tests: hand-built `ServiceCollection` providers that resolve `IToolGateway` must call `services.AddLogging()` (the gateway now depends on `ILogger<ToolGateway>`). Direct `new ToolGateway(registry, provider, logger)` construction can pass any `ILogger<ToolGateway>` (the telemetry tests use a recording logger).
- No PowerShell changed this slice, so no ASCII/parser-validation step was required.

status: telemetry merged 2026-05-21. tests 231 passing (was 227, +4). gateway invocations now emit one structured ToolGatewayInvocation log each; outbound header injection deferred to the analyze.sports wrap. no FastAPI/prompt/confidence/CognitiveProtocolBuilder/DB/Angular/MCP/Azure/pgvector changes. jera-workspace-skills untouched.
