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

## addendum: Tool Gateway Analyze Wrap v1 (2026-05-21)

Runtime slice. Routes the internal .NET -> FastAPI sports analysis call through the Tool Gateway. After this slice the analyze stage joins reference and retrieve behind the gateway; the only remaining direct external/internal call paths are gone from the pipeline. No FastAPI implementation, prompt, Pydantic-contract-name, CognitiveProtocolBuilder, confidence-rule, DB, Angular, MCP, Azure Functions, pgvector, or Container Apps changes.

### naming and skills gate

Skills: `dai-grill-with-vault` (read FastApiClient, SportsAnalyzer, the request/response contracts, and the consumption path before naming), `dai-token-tight` (reporting), `superpowers:test-driven-development` (RED 40 errors -> GREEN), `superpowers:verification-before-completion` (fresh suite + named-test confirmation), `dai-agent-handoff` shaping. Skill-fit note carried forward, unchanged: a dedicated `dai-audit`/`dai-implement-with-vault` skill would suit solo implementation slices better than the interactive grill template. jera-workspace-skills untouched (no approval to edit).

Naming decision (reviewed the four candidates; confirmed the preferred):
- **`analysis.sports.matchup_read`** chosen. `analysis` is a durable noun-domain (parallel to market/schedule/pitching). `sports` honestly signals one tool across all sport families (the analyze call is family-agnostic at this layer; FastAPI dispatches by competition internally). `matchup_read` names the product output ("the read" / "Read Stance") without leaking implementation.
- Rejected: `analysis.sports.matchup_analysis` (analysis/analysis stutter); `analyze.sports` (verb-domain, thin on capability, less self-describing); `model.sports.matchup_analysis` (`model` ties to implementation -- would mislead if analyze ever became partly deterministic).
- Protocol node constant added: **`ProtocolNodes.PlatformAnalyze = "platform.analyze"`** -- parallels `platform.reference` and `platform.retrieve`. The caller is platform orchestration (SportsAnalyzer); the 12 cognitive micro-actions are produced inside the model call, but the tool *caller* is the platform analyze stage, so a platform-stage sentinel is correct.
- Helper/type names: `ToolIds.AnalysisSportsMatchupRead`, `AnalysisSportsMatchupReadHandler`, `SportsMatchupReadInput(SportsAnalysisRequest Request, Guid AgentRunId)`. AgentRunId is an explicit input field (a functional FastAPI argument forwarded as X-Agent-Run-Id), not read from context, to avoid depending on a nullable.
- Registry metadata: `Kind.Analyzer`, `Transport.HttpInternal`, `Idempotency.PerRun`, `CacheTtl = null` (each run analyzes fresh), `CostClass.ModelCall`. All enum values already existed.

### files changed

- `dai/platform/dotnet/DevCore.Api/Tools/ToolInvocationContext.cs` -- `ProtocolNodes.PlatformAnalyze`.
- `dai/platform/dotnet/DevCore.Api/Tools/ToolRegistry.cs` -- `ToolIds.AnalysisSportsMatchupRead` + one `ToolDefinition` (Analyzer/HttpInternal/PerRun/no-cache/ModelCall/PlatformAnalyze).
- `dai/platform/dotnet/DevCore.Api/Tools/Handlers/AnalysisSportsMatchupReadHandler.cs` (new) -- `SportsMatchupReadInput` + handler wrapping `FastApiClient.AnalyzeSportsMatchupAsync` 1:1.
- `dai/platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- keyed-scoped handler registration.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsAnalyzer.cs` -- constructor takes `IToolGateway` instead of `FastApiClient`; builds the request as before, then routes the single call through the gateway with a `platform.analyze` context (`RunId = artifact.AgentRunId`, `CorrelationId = Activity.Current?.Id`).
- `dai/platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayAnalyzeTests.cs` (new) -- dispatch (allowed), denied node, FastAPI-failure-rethrow.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsAnalyzerTests.cs` -- `SportsAnalyzer` built with a real gateway wired to a capturing FastApiClient; asserts request-body shape, response mapping, AND that `X-Agent-Run-Id` / `X-Correlation-Id` outbound headers are preserved through the gateway.
- Program.cs unchanged: `AddDaiToolGateway()` registers the gateway + handler; `FastApiClient` stays registered via `AddHttpClient<FastApiClient>` (now consumed by the handler, not SportsAnalyzer).

### behavior summary

No behavior change. The analyze call now flows `SportsAnalyzer -> IToolGateway -> AnalysisSportsMatchupReadHandler -> FastApiClient.AnalyzeSportsMatchupAsync`. FastApiClient still owns the wire contract, the SportsAnalysisResponse mapping, and the `X-Correlation-Id` / `X-Agent-Run-Id` headers (which already existed and are unchanged). Legacy `CognitivePhases` and the canonical `CognitiveProtocolBuilder` path are downstream of the response and untouched. A FastAPI failure (non-2xx) still throws the same exception via `EnsureSuccessStatusCode`; the gateway logs it as `HandlerError` and re-throws, so the existing `RecordAnalyzeFailed`/`ComposeFailedRun` pipeline behavior is preserved. The gateway enforces `platform.analyze` and fails closed for any other caller. Telemetry now emits a `ToolGatewayInvocation` event for the analyze call with `CostClass=ModelCall`.

Outbound correlation headers: already present in `FastApiClient` (added before this work); the wrap preserves them. No new header injection was needed -- scope item 8 satisfied by existing code and verified by the header-preservation test.

### test results

`dotnet test`: 234 passed, 0 failed (was 231, +3). New/updated: `invoke_dispatches_analysis_sports_matchup_read_to_fastapi_when_node_is_allowed`, `invoke_throws_tool_not_allowed_when_matchup_read_called_from_non_analyze_node`, `invoke_preserves_fastapi_failure_as_thrown_exception`, and `analyze_forwards_sharp_public_context_and_correlation_headers_through_gateway` (replaces the prior direct-client analyzer test). WebApplicationFactory integration tests pass, confirming the gateway-routed analyze resolves through the real Program.cs DI graph.

### risks

- `SportsAnalyzer` now depends solely on the gateway; a dropped registration fails the analyze stage closed (covered by the integration tests + the analyze dispatch tests).
- `AgentRunId` is duplicated (context.RunId and input.AgentRunId) -- same value, intentional: context.RunId is for telemetry, input.AgentRunId is the functional FastAPI argument. Documented in the handler.
- Telemetry classifies any analyze exception as `HandlerError` (catch-all), then re-throws; behavior unchanged, classification coarse.
- Idempotency `PerRun` on the analyze tool is declarative only; there is no run-scoped analyze cache yet (one call per run already, so no duplicate today).

### next slice

The full sports pipeline (reference, retrieve, analyze) now routes through the gateway. Natural next steps: (a) Container Apps deploy slice (package both services, internal DNS, Key Vault, App Insights, smoke parity) now that the gateway seam and telemetry are in place; or (b) gateway idempotency-cache enforcement reading the registry `Idempotency`/`CacheTtl` fields (currently declarative). Recommend (a) -- the launch target -- since the wrap work that made the system observable and governable is complete.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (7 files: 3 new, 4 modified) and `dai-vault` (this handoff). `jera-workspace-skills` read-only, untouched.
- Re-verify anywhere: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 234 passing. No stack/DB/network needed (fake HTTP handlers throughout).
- Extension pattern is identical to prior wraps: `ToolDefinition` + `ToolIds` constant, `IToolHandler<TInput, TOutput>` in `Tools/Handlers/`, keyed-scoped registration, route the call site through `gateway.InvokeAsync(...)` with the correct `ProtocolNodes.*`. For internal calls use `Transport.HttpInternal`; the analyze tool is `Kind.Analyzer` + `CostClass.ModelCall`.
- Gotcha: hand-built `ServiceCollection` test providers that resolve `IToolGateway` must call `services.AddLogging()` (the gateway depends on `ILogger<ToolGateway>` since the telemetry slice).
- No PowerShell changed this slice, so no ASCII/parser-validation step was required.

status: analyze wrap merged 2026-05-21. tests 234 passing (was 231, +3). the sports pipeline's reference, retrieve, and analyze calls all route through the Tool Gateway now. outbound X-Agent-Run-Id / X-Correlation-Id headers preserved (already in FastApiClient). no FastAPI/prompt/Pydantic/CognitiveProtocolBuilder/confidence/DB/Angular/MCP/Azure/pgvector changes. jera-workspace-skills untouched.

## addendum: Cloud Deploy Readiness v1 (2026-05-21)

Docs-only readiness slice. No deployment, no runtime code, no config change. Produced the assessment that tells the containerize slice exactly what to build.

### naming and skills gate

Skills: `dai-grill-with-vault` (read Program.cs, appsettings, agent-service main.py, sports-app environments, and the dev scripts before naming or asserting anything), `dai-token-tight` (chat reporting; the doc itself is a vault artifact and uses full prose), `superpowers:verification-before-completion` (every readiness claim grounded in a file read; em-dash check on the new doc). No TDD (docs only, no testable code). jera-workspace-skills untouched (no approval to edit). Skill-fit note carried forward, unchanged: a dedicated `dai-audit`/`dai-implement-with-vault` skill would fit solo read-and-assess slices better than the interactive grill template.

Naming decisions:
- Container app names: `dai-api` (DevCore.Api orchestrator), `dai-analyzer` (FastAPI agent-service), `dai-sports-web` (Angular sports-app). Lowercase, hyphenated, product-prefixed, role-suffixed; map cleanly to code. Azure SQL is managed, not a container app.
- Env vars use the standard `Section__Key` double-underscore convention for .NET (`ConnectionStrings__Sql`, `AiService__BaseUrl`, `OddsApi__ApiKey`, `Dev__EnableBypassAuth`, `AzureAd__TenantId/Audience`, `APPLICATIONINSIGHTS_CONNECTION_STRING`); `OPENAI_API_KEY` for the analyzer.
- Doc filename: `cloud-deploy-readiness-v1.md` (preferred).

### files changed

- `dai-vault/02 Platform/architecture/cloud-deploy-readiness-v1.md` (new) -- 12-section readiness assessment: inventory, local shape, target ACA shape, env vars, secrets/Key Vault, health, networking, logging, launch-required, post-launch, blockers, next slice.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No `dai` repo changes.

### readiness summary

Three deployable services plus managed SQL. Target: `dai-api` (public ingress) + `dai-analyzer` (internal-only) in one Container Apps environment, `dai-sports-web` as a static build, Azure SQL managed. The Tool Gateway's structured `ToolGatewayInvocation` telemetry is the observability surface; wire App Insights into both containers. Internal call stays `POST /api/sports/analyze` with the existing correlation headers; only the hostname changes from `127.0.0.1:8000` to `http://dai-analyzer`. Corrected a stale detail: the analyzer port is 8000, not 8001 as an earlier draft of the cloud plan said.

### launch blockers

1. **Committed secrets (security, top priority).** Real SQL password in `appsettings.json` and `appsettings.Development.json`; real `OddsApi:ApiKey` and a `Dev:ProvisionKey` in `appsettings.Development.json`. They are in git history, so they must be rotated, not just deleted. This readiness slice flags them; it does not edit the files.
2. No Dockerfiles for any service.
3. `Dev:EnableBypassAuth = true` must be `false` in cloud; real customer auth (Entra External ID) must precede public exposure.
4. No `/health` endpoint on `dai-api`.
5. Angular production `apiBaseUrl` hardcoded to `localhost:5007`.
6. No managed Azure SQL provisioned.
7. `dai-api` CORS allows dev origins only.

### next slice

1. **Secrets hygiene (urgent, first):** rotate the committed SQL password and OddsApi key, remove real values from `appsettings*.json`, move local dev to user secrets / gitignored `.env`.
2. **Containerize:** author the three Dockerfiles, add the `dai-api` `/health` endpoint, externalize config to env vars (dev behavior unchanged because `appsettings.Development.json` still supplies values), verify local container builds + smoke parity. Azure provisioning follows.

### Claude <-> Codex transfer notes

- Docs-only slice. `dai-vault` has one new doc + this handoff. `dai` and `jera-workspace-skills` untouched.
- The readiness doc is the build spec for the containerize slice; start there.
- Do NOT commit any rotated secret value. Secret rotation happens in the secrets-hygiene slice and the new values go to Key Vault / user secrets, never to tracked files.
- No PowerShell changed this slice; no ASCII/parser-validation step required.

status: readiness assessed 2026-05-21. docs only. top blocker is committed secrets in appsettings (rotate + externalize). next: secrets hygiene, then containerize. no code, no deploy. jera-workspace-skills untouched.

## addendum: Secrets Hygiene v1 (2026-05-21)

Security slice. Removed the one real secret from tracked config and added an onboarding template. No git history rewrite, no credential rotation performed (rotation is a manual step for the human, checklist below), no runtime behavior change.

### naming and skills gate

Skills: `dai-grill-with-vault` (read appsettings, csproj, .gitignore, .env.example, and ran `git ls-files` / `git check-ignore` / `git log` to verify tracked-vs-ignored state before changing anything), `dai-token-tight` (reporting), `superpowers:verification-before-completion` (proved tracked state and the post-change clean state with commands, not assumptions). No TDD authored, but ran the full suite because a base-config change could affect startup. jera-workspace-skills untouched (no approval). Skill-fit note carried forward, unchanged.

Naming decisions: placeholder token `__REPLACE_VIA_USER_SECRETS_OR_ENV__` for the SQL password, `__REPLACE_VIA_USER_SECRETS__` for the OddsApi key / Dev provision key, `__REPLACE_WITH_TENANT_ID__` / `api://__REPLACE_WITH_API_APP_ID__` for AzureAd identifiers. Example file `appsettings.Development.example.json` (mirrors the `.env.example` convention; not caught by the `**/appsettings.Development.json` gitignore rule, confirmed via `git check-ignore`).

### findings (verified, correcting the readiness draft)

- Only `appsettings.json` (base) was a tracked secret: a real SQL password in `ConnectionStrings:Sql`. Genuine git-history exposure.
- `appsettings.Development.json` is gitignored (`**/appsettings.Development.json`) and was NEVER committed (empty `git log`). Its dev SQL password, `OddsApi:ApiKey`, and `Dev:ProvisionKey` are NOT in the repo or history; they live only in the developer's local gitignored copy. The readiness doc's "committed in both files / in git history" claim was overstated and has been corrected in `cloud-deploy-readiness-v1.md`.
- `services/agent-service/.env` gitignored; `.env.example` keyless. `CompetitionCatalog.cs` `OddsApiKey` values are public odds-api sport-route slugs (e.g. `americanfootball_nfl`), not the secret API key. `DevProvisionController` reads `Dev:ProvisionKey` from config (no hardcoded value). Test files use `"test-key"`.

### files changed

- `dai/platform/dotnet/DevCore.Api/appsettings.json` -- SQL password replaced with `__REPLACE_VIA_USER_SECRETS_OR_ENV__`; all non-secret structure preserved.
- `dai/platform/dotnet/DevCore.Api/appsettings.Development.example.json` (new, tracked) -- onboarding template: real non-secret dev defaults (`AiService:BaseUrl = http://127.0.0.1:8000`), placeholders for secrets, header comment with the `dotnet user-secrets set` commands.
- `dai-vault/02 Platform/architecture/cloud-deploy-readiness-v1.md` -- corrected the urgent-finding and blocker #1 wording to the verified tracked-vs-ignored reality.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).
- Not changed: `appsettings.Development.json` (untracked, the developer's local real values stay there, gitignored); `.gitignore` (already covers `.env`, `.env.*`, `**/appsettings.Development.json`, `services/agent-service/.env`); `Program.cs` (csproj already has a `UserSecretsId`, so user secrets auto-load in Development with no code change); `DevCore.Api.csproj`.

### secrets removed from tracked config

- SQL Server password in `appsettings.json` `ConnectionStrings:Sql` -> placeholder. This is the only secret that was in tracked config.

### manual rotation checklist (human action, outside Claude)

- [ ] **SQL password** -- rotate the `sa` (or app) SQL credential; it was in committed git history. Update the local `appsettings.Development.json` or user secrets with the new value, and set `ConnectionStrings__Sql` in Container Apps / Key Vault for cloud.
- [ ] **Odds API key** -- never committed, but surfaced in session transcripts; rotate at api.the-odds-api.com as defense-in-depth and set via `dotnet user-secrets set "OddsApi:ApiKey" "<new>"` locally and Key Vault for cloud.
- [ ] **Dev provision key** -- never committed; regenerate the local value if desired; not used in cloud (a dev-only bypass path).
- [ ] **OpenAI API key** -- lives in the gitignored `services/agent-service/.env`; not tracked. Rotate only if otherwise exposed; set `OPENAI_API_KEY` in Key Vault for cloud.
- [ ] Optional hardening: history rewrite (BFG / git filter-repo) to purge the old SQL password from history. Out of scope for this slice (no history modification was performed); rotation makes the historical value useless, which is the pragmatic fix.

### local dev configuration approach

- The developer's existing `appsettings.Development.json` is untracked and unchanged, so current local dev is unaffected.
- A fresh clone has no `appsettings.Development.json`. Two supported paths: (a) `cp appsettings.Development.example.json appsettings.Development.json` and fill in real values (stays gitignored), or (b) leave no dev file and set secrets via `dotnet user-secrets set` (the csproj `UserSecretsId` makes user secrets auto-load in Development). Either way, no secret is tracked.

### cloud deploy implications

- Cloud supplies every secret via env vars / Key Vault: `ConnectionStrings__Sql`, `OddsApi__ApiKey`, `OPENAI_API_KEY`, and (non-secret config) `AiService__BaseUrl`, `AzureAd__*`, `Dev__EnableBypassAuth=false`. The base `appsettings.json` placeholder is never used in cloud because the env var overrides it.

### verification results

- `git ls-files` / `git check-ignore` / `git log`: confirmed base `appsettings.json` tracked with the secret; `appsettings.Development.json` ignored and never committed.
- `git grep "Password=" -- '*.json'` after the change: only the placeholder line in `appsettings.json` remains. No real password in tracked json. ApiKey/ProvisionKey: no real values in tracked json.
- `git check-ignore appsettings.Development.example.json`: not ignored (trackable).
- `dotnet test`: 234 passed, 0 failed (base-config change is startup-safe; the test factory uses an in-memory DB so the placeholder connection string is never opened).
- No PowerShell changed, so no ASCII/parser-validation step was required.

### risks

- The old SQL password remains in git history until rotated (no history rewrite this slice). Rotation neutralizes it; tracked-tree exposure is removed now.
- A fresh clone without a `appsettings.Development.json` or user secrets will fail to connect to SQL until one is provided -- intended secure default, documented above.
- The OddsApi key / provision key were exposed in session transcripts (not git); rotation is prudent.

### next slice

Containerize: three Dockerfiles, the `dai-api` `/health` endpoint, externalize config to env vars, verify local container builds + smoke parity. With tracked secrets removed, this is unblocked. (Manual SQL rotation should happen in parallel; it does not block authoring Dockerfiles but must precede a real deploy.)

### Claude <-> Codex transfer notes

- `dai` changed: `appsettings.json` (placeholdered) + new `appsettings.Development.example.json`. `dai-vault`: readiness doc correction + this handoff. `jera-workspace-skills` untouched.
- Do NOT commit `appsettings.Development.json` (gitignored) or any rotated secret value. The example file holds placeholders only.
- Re-verify: `git grep "Password=" -- '*.json'` should show only the placeholder; `dotnet test` -> 234 passing.
- No history rewrite was performed; the SQL password rotation is a manual human step (checklist above).

status: secrets hygiene merged 2026-05-21. tracked SQL password placeholdered; onboarding example added; readiness doc corrected. 234 tests passing. SQL credential rotation is a pending manual human action. no history rewrite, no rotation performed by Claude. jera-workspace-skills untouched.

## addendum: Containerization Readiness v1 (2026-05-21)

Build-readiness slice. Added the `dai-api` `/health` endpoint (TDD) and Dockerfiles for `dai-api` and `dai-analyzer`, and verified both images build and that `dai-api` serves `/health` from the container. No Azure resources created. No FastAPI prompt, Pydantic, CognitiveProtocolBuilder, confidence, DB schema, Angular, MCP, pgvector, Azure Functions, or Kubernetes changes.

### naming and skills gate

Skills: `dai-grill-with-vault` (read Program.cs, the csproj graph, requirements.txt, the readiness doc, and confirmed Docker availability before authoring), `dai-token-tight` (reporting), `superpowers:test-driven-development` (RED 404 -> GREEN for `/health`), `superpowers:verification-before-completion` (ran the full suite, both `docker build`s, and a container `/health` smoke). jera-workspace-skills untouched (no approval). Skill-fit note carried forward, unchanged: a dedicated `dai-audit`/`dai-implement-with-vault` skill would suit these solo build slices.

Naming decisions:
- Health endpoint `GET /health` for `dai-api` (boring, standard; liveness only); reuse existing `GET /api/ping` for `dai-analyzer` (no new endpoint).
- Dockerfile locations: `platform/dotnet/Dockerfile` (build context is the .NET solution root because DevCore.Api references Domain/Data/AiClient; build with `docker build -t dai-api platform/dotnet`) and `services/agent-service/Dockerfile` (`docker build -t dai-analyzer services/agent-service`). A `.dockerignore` sits in each context.
- Container app names unchanged from the readiness doc: `dai-api`, `dai-analyzer`, `dai-sports-web`.
- Container internal ports: `dai-api` on 8080 (`ASPNETCORE_URLS=http://+:8080`, the aspnet image default), `dai-analyzer` on 8000 (uvicorn). The internal `AiService__BaseUrl` points dai-api at `http://dai-analyzer:8000` in cloud.

### files changed

- `dai/platform/dotnet/DevCore.Api/Program.cs` -- added `app.MapGet("/health", () => Results.Ok(new { status = "ok" }))` after `MapControllers()`. Anonymous, no DB.
- `dai/platform/dotnet/DevCore.Api.Tests/Integration/HealthEndpointTests.cs` (new) -- `health_endpoint_returns_ok_without_auth` via WebApplicationFactory (in-memory DB swap + identity stub, established pattern).
- `dai/platform/dotnet/Dockerfile` (new) -- multi-stage .NET build (sdk:10.0 -> aspnet:10.0), csproj-graph restore for layer caching, publishes DevCore.Api. No secrets baked in.
- `dai/platform/dotnet/.dockerignore` (new) -- excludes bin/obj, the test project, `appsettings.Development.json`, logs.
- `dai/services/agent-service/Dockerfile` (new) -- python:3.12-slim, pip install requirements, uvicorn on 0.0.0.0:8000. OPENAI_API_KEY supplied at runtime, not baked.
- `dai/services/agent-service/.dockerignore` (new) -- excludes .venv, __pycache__, `.env`/`.env.*` (keeps `.env.example`), tests.
- `dai-vault/02 Platform/architecture/cloud-deploy-readiness-v1.md` -- marked the Dockerfile and `/health` blockers resolved; health section updated.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

### container readiness summary

`dai-api` and `dai-analyzer` are now container-buildable from the repo. The Tool Gateway and CognitiveProtocol paths are not bypassed: the .NET image publishes DevCore.Api unchanged (gateway DI, telemetry, and the analyze wrap are all in the published assembly), and the analyzer image runs the same FastAPI app. `dai-sports-web` remains a static build (Angular), hosted on Static Web Apps / Storage+CDN per the readiness doc; its production `apiBaseUrl` must be set to the deployed `dai-api` URL at build time (still an open launch-required item, unchanged).

### health check behavior

- `dai-api`: `GET /health` -> 200 `{"status":"ok"}`, anonymous, no database dependency. Verified live from the built container (`docker run` + curl).
- `dai-analyzer`: existing `GET /api/ping` is the liveness probe.

### environment variable map

- `dai-api` (container, `Section__Key`): `ASPNETCORE_ENVIRONMENT=Production`, `ASPNETCORE_URLS=http://+:8080` (set in image), `ConnectionStrings__Sql` (Key Vault), `AiService__BaseUrl=http://dai-analyzer:8000`, `AzureAd__TenantId`, `AzureAd__Audience`, `OddsApi__ApiKey` (Key Vault), `Dev__EnableBypassAuth=false`, `APPLICATIONINSIGHTS_CONNECTION_STRING` (Key Vault).
- `dai-analyzer` (container): `OPENAI_API_KEY` (Key Vault), uvicorn host/port set in image CMD.
- `dai-sports-web`: `apiBaseUrl` is build-time (environment.ts), not a runtime env var.

### verification results

- `dotnet test`: 235 passed, 0 failed (was 234, +1 health test). RED confirmed first (404), then GREEN.
- `docker build -t dai-analyzer services/agent-service`: success (image 318MB).
- `docker build -t dai-api platform/dotnet`: success; all four .NET projects compiled, DevCore.Api published (image 374MB).
- container smoke: `docker run dai-api` then `GET /health` -> 200 `{"status":"ok"}`; container removed.
- No PowerShell changed, so no ASCII/parser-validation step was required.

### risks

- Images build and `dai-api` serves health locally; full end-to-end in-container run (dai-api -> dai-analyzer over a docker network, with a real SQL and OPENAI key) was not exercised this slice. That belongs to the first actual deploy / a compose-based local integration check.
- `dai-api` health is liveness-only; it does not check SQL or analyzer reachability. A readiness probe with dependency checks is a deliberate later concern.
- Base image tags are floating `10.0` / `3.12-slim`; pin to digests or patch tags before production for reproducibility.
- The SQL credential rotation from Secrets Hygiene v1 is still a pending human action and gates a real deploy (not this slice).

### next slice

Azure Container Apps provisioning: create the Container Apps environment, push images to a registry (ACR), wire internal ingress (`dai-analyzer` internal-only) and `AiService__BaseUrl`, set env vars / Key Vault refs, point `dai-sports-web` at the deployed `dai-api` URL, set `Dev__EnableBypassAuth=false`. Gated on the manual SQL rotation and on customer auth (Entra External ID) for public exposure.

### Claude <-> Codex transfer notes

- `dai` changed: Program.cs (+/health), one new test, two Dockerfiles, two .dockerignore. `dai-vault`: readiness doc updates + this handoff. `jera-workspace-skills` untouched.
- Reproduce builds: `docker build -t dai-api platform/dotnet` and `docker build -t dai-analyzer services/agent-service`. Smoke: `docker run -d -p 8090:8080 dai-api` then `curl localhost:8090/health` -> 200.
- Re-verify tests: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 235 passing.
- No secrets are in the images (.dockerignore excludes `.env` and `appsettings.Development.json`; no secret ENV baked). Runtime config is injected by Container Apps.
- No PowerShell changed this slice.

status: containerization readiness merged 2026-05-21. dai-api /health added (TDD); dai-api + dai-analyzer Dockerfiles build clean; /health verified from container. 235 tests passing. no Azure deploy. SQL rotation + customer auth still gate a real deploy. jera-workspace-skills untouched.

## addendum: Local Container Compose Smoke v1 (2026-05-21)

Local-only smoke slice. Added a docker compose file that builds and runs `dai-api` + `dai-analyzer` together and proves container-to-container parity by service name, mirroring the Container Apps internal-DNS shape. No Azure resources, no .NET code change, no secrets committed. FastAPI prompts / Pydantic / CognitiveProtocolBuilder / confidence / DB schema / Angular / MCP / pgvector / Azure Functions / Kubernetes all untouched.

### naming and skills gate

Skills: `dai-grill-with-vault` (read the Dockerfiles, the analyzer `/api/ping` route, compose availability, and the readiness doc before authoring), `dai-token-tight` (reporting), `superpowers:verification-before-completion` (ran `docker compose config`, `up --build`, host + cross-service smoke, and teardown -- evidence, not assumption). No TDD (no testable code; compose is config). jera-workspace-skills untouched (no approval). Skill-fit note carried forward, unchanged.

Naming decisions:
- Compose file `compose.smoke.yaml` at the `dai/` repo root (must reference both build contexts). The `.smoke` qualifier and a header comment make "local smoke, not production" explicit; Compose v2 `compose.*.yaml` spec name.
- Compose project name `dai-smoke`; network `dai-smoke` (bridge). Service names `dai-api` and `dai-analyzer` match the launch container-app names.
- Internal URL env on dai-api: `AiService__BaseUrl=http://dai-analyzer:8000` (service-name DNS).

### files changed

- `dai/compose.smoke.yaml` (new) -- two services (`dai-api` build context `./platform/dotnet`, `dai-analyzer` build context `./services/agent-service`), a `dai-smoke` bridge network, published ports 8080 and 8000, and `AiService__BaseUrl=http://dai-analyzer:8000`. No secrets; SQL and `OPENAI_API_KEY` intentionally unset with a documented rationale.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum). `cloud-deploy-readiness-v1.md` needed no correction.

### local compose / smoke behavior

`docker compose -f compose.smoke.yaml up -d --build` builds both images and starts them on the `dai-smoke` bridge. dai-api reaches dai-analyzer by the service name `dai-analyzer` exactly as it will in Container Apps internal ingress. SQL is excluded by design: the smoke checks do not touch the database, so no connection string or model key is needed, keeping the smoke secret-free. The full analyze chain (SQL + OpenAI) is a deploy/integration concern, not this smoke.

### health check results

- Host `GET http://localhost:8080/health` -> 200 `{"status":"ok"}`.
- Host `GET http://localhost:8000/api/ping` -> 200 `{"status":"ok"}`.

### internal service communication result

- Cross-service `dai-analyzer -> http://dai-api:8080/health` (python urllib inside the analyzer container) -> 200, proving compose service-name DNS + reachability.
- Throwaway client on the `dai-smoke` network -> `http://dai-analyzer:8000/api/ping` -> 200, proving the exact `AiService__BaseUrl` target dai-api uses resolves and responds.
- Teardown (`docker compose down`) removed both containers and the network cleanly.

### environment variable notes

- compose sets only non-secret values: `ASPNETCORE_ENVIRONMENT=Production`, `ASPNETCORE_URLS=http://+:8080`, `AiService__BaseUrl=http://dai-analyzer:8000`.
- `ConnectionStrings__Sql` and `OPENAI_API_KEY` are intentionally unset for the smoke. For a fuller local run, supply them via shell env or a gitignored `.env` (compose reads `.env`); never commit them. dai-api boots on its baked placeholder connection string because `/health` never opens a connection.

### risks

- The smoke covers build + boot + health + service-name reachability. It does NOT exercise the full analyze chain (requires SQL + OpenAI key) -- that is the first real deploy / a separate integration check.
- Published host ports 8080/8000 may collide with a running local dev stack (.NET dev uses 5007, FastAPI dev uses 8000). Run the compose smoke when the dev FastAPI is stopped, or remap the host port. Documented for the next runner.
- Base image tags still float (`10.0`, `3.12-slim`); pin before production.
- SQL rotation and customer auth remain the gating manual blockers for a real deploy (unchanged).

### next slice

Azure Container Apps provisioning: push images to ACR, create the Container Apps environment, set `dai-analyzer` internal-only ingress with `AiService__BaseUrl` pointed at its internal FQDN, wire Key Vault refs, `Dev__EnableBypassAuth=false`, and point `dai-sports-web` at the deployed `dai-api`. Gated on the manual SQL rotation and customer auth.

### Claude <-> Codex transfer notes

- `dai` changed: one new file `compose.smoke.yaml`. No code, no test changes. `dai-vault`: this handoff. `jera-workspace-skills` untouched.
- Reproduce: `cd dai && docker compose -f compose.smoke.yaml up -d --build`, then `curl localhost:8080/health` and `curl localhost:8000/api/ping` (both 200), then `docker compose -f compose.smoke.yaml down`. Stop the local dev FastAPI first if port 8000 is busy.
- No .NET code changed this slice, so the test suite is unaffected (235 passing from Containerization Readiness v1). No PowerShell changed; no ASCII/parser step.

status: local compose smoke merged 2026-05-21. compose.smoke.yaml builds + runs dai-api and dai-analyzer; /health, /api/ping, and service-name reachability all verified 200; clean teardown. no Azure, no secrets, no code change. jera-workspace-skills untouched.

## addendum: Azure Container Apps Provisioning Plan v1 (2026-05-22)

Docs-only planning slice. Produced the precise, named Azure Container Apps provisioning plan that the create slice executes against. No Azure resources created. No runtime code, no FastAPI prompt, no Pydantic contract, no CognitiveProtocolBuilder mapping, no confidence rule, no DB schema, no Angular behavior, no MCP, no pgvector, no Azure Functions, no Kubernetes change.

### naming and skills gate

Skills: `dai-grill-with-vault` (read Program.cs auth/CORS, both Dockerfiles, appsettings + the example template, the sports-app environments, compose.smoke.yaml, and both cloud docs before naming or asserting), `dai-token-tight` (chat reporting; the doc itself is a vault artifact and uses full prose), `dai-agent-handoff` (shaped these transfer notes), `superpowers:verification-before-completion` (every claim grounded in a file read with path:line; ASCII check run on the new doc). No TDD (docs only, no testable code). `dai-signal-follow-up-diagnostics` considered and skipped (it diagnoses run signal gaps, not infra). jera-workspace-skills untouched (no approval to edit). Skill-fit note carried forward, unchanged: a dedicated `dai-audit`/`dai-implement-with-vault` skill would fit solo read-and-plan slices better than the interactive grill closing template.

Naming review result (item documented per gate): the three Container Apps keep the doctrine-locked names `dai-api`, `dai-analyzer`, `dai-sports-web` and deliberately do NOT take the CAF `ca-` prefix, because the internal service URL contract depends on the app name (`http://dai-analyzer`) and the names are already locked across compose, both Dockerfiles, and the readiness doc. Every other resource follows CAF abbreviations with a `dai`/`prod` token: `rg-dai-prod`, `crdaiprod` (ACR forbids hyphens), `cae-dai-prod`, `kv-dai-prod`, `log-dai-prod`, `appi-dai-prod`, `sql-dai-prod` + db `devcore` (kept to avoid connection-string drift), `swa-dai-sports-web` (Static Web Apps, the chosen static-hosting alternative), optional `id-dai-api`. Key Vault secret names are hyphenated (vault forbids underscores) and mapped to the `Section__Key` env vars: `connectionstrings-sql`, `oddsapi-apikey`, `appinsights-connection-string`, `openai-api-key`. All names lowercase, product-prefixed, stable, boring; no misleading names found.

### files changed

- `dai-vault/02 Platform/architecture/azure-container-apps-provisioning-plan-v1.md` (new) -- 20-section provisioning plan: naming, RG, region + open question, ACR images, environment, app names, ingress, internal URL, env vars, Key Vault secrets, App Insights/Log Analytics, Azure SQL, CORS, Entra External ID gating, build/push sequence, smoke checklist, rollback, manual pre-deploy blockers, deferred items, next slice. ASCII-clean (verified).
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No `dai` repo changes. No `jera-workspace-skills` changes.

### provisioning plan summary

One resource group `rg-dai-prod`, one region (recommend `eastus2`, open question pending customer/latency/Entra-residency signal), one Container Apps environment `cae-dai-prod` backed by `log-dai-prod`. Two images in `crdaiprod`: `dai-api` (public ingress, 8080) and `dai-analyzer` (internal-only ingress, 8000); Angular ships as Static Web Apps `swa-dai-sports-web`, not a container. `AiService__BaseUrl=http://dai-analyzer` (no port; ACA internal ingress maps port 80 to target 8000, differs from the compose `:8000`). Four secrets in `kv-dai-prod` as Key Vault refs; non-secret config (AzureAd, AiService BaseUrl, env, bypass flag) as plain env. App Insights `appi-dai-prod` ingests the existing `ToolGatewayInvocation` structured logs; correlate on `X-Agent-Run-Id`. Azure SQL `sql-dai-prod`/`devcore`, no schema change, connection string from Key Vault. Images tagged with immutable git sha so rollback is a revision pin. Tool Gateway, CognitiveProtocol persistence, and the analyze wrap ship unchanged inside the images.

### manual blockers (human, before a real deploy)

1. SQL password rotation (gating) -- historical value still live in git history; rotate, then store the new connection string as Key Vault `connectionstrings-sql`.
2. Odds API key rotation (defense-in-depth; surfaced in transcripts, never committed) -> Key Vault `oddsapi-apikey`; rotate OpenAI key similarly -> `openai-api-key`.
3. Customer auth decision (gating for public exposure) -- stand up Entra External ID (tenant, user flow, API + SPA app registrations, audience) and confirm the JWT authority shape; the current `Program.cs:47` authority is the workforce Entra form, External ID uses a different host. Public `dai-api` must not go live with `Dev__EnableBypassAuth=true` or without working customer identity.

Two small code prerequisites for the create slice (gated by the above): externalize CORS allowed origins to config (currently hardcoded `Program.cs:22-26`), and confirm/adjust the JWT authority for External ID.

### next recommended slice

Azure Container Apps provisioning execution (the create slice): provision the named resources, populate Key Vault with rotated secrets, stand up Azure SQL, build/push both images, create the environment, deploy `dai-analyzer` (internal) then `dai-api` (public) with the section 9 env vars and `AiService__BaseUrl=http://dai-analyzer`, externalize CORS + deploy `swa-dai-sports-web`, then run the section 16 smoke checklist. Gated on the manual blockers; provisioning can proceed up to but not including public exposure of `dai-api`.

### Claude <-> Codex transfer notes

- Docs-only slice. `dai-vault`: one new doc (`azure-container-apps-provisioning-plan-v1.md`) + this handoff. `dai` and `jera-workspace-skills` untouched.
- The new doc is the build spec for the provisioning execution slice; start there. It cites the grounding files (Program.cs auth/CORS lines, Dockerfiles, env templates, sports-app environments).
- Do NOT commit any rotated secret value. New secrets go to Key Vault only.
- Pre-existing untracked items in `dai-vault` (the 20260508/20260509/20260510 nba calibration md + artifacts json under `04 Products/sports-v1/calibration/`) are NOT from this slice; leave them.
- No PowerShell or .NET code changed; no ASCII/parser step beyond the ASCII check on the new vault doc (clean).

status: provisioning plan written 2026-05-22. docs only, no Azure resources created. names reviewed (container apps doctrine-locked, rest CAF). manual blockers: SQL rotation, odds/openai key rotation, Entra External ID decision. next: provisioning execution slice. jera-workspace-skills untouched.

## addendum: Customer Auth Readiness v1 (2026-05-22)

Docs-only auth readiness slice. Clarified the customer identity plan (Entra External ID) before public exposure and removed the workforce-vs-External-ID ambiguity in the deploy docs. No auth code implemented. No Azure resources created. No FastAPI prompt, Pydantic contract, CognitiveProtocolBuilder mapping, confidence rule, DB schema, Angular product behavior, MCP, pgvector, Azure Functions, or Kubernetes change. No secrets committed. Tool Gateway, CognitiveProtocol persistence, and local dev workflow preserved.

### naming and skills gate

Skills: `dai-grill-with-vault` (read Program.cs auth/CORS, IdentityResolver, DevProvisionController, UserIdentity model, sports-app app.config.ts, and the auth section of next-platform-architecture-plan.md + both cloud docs before asserting or naming), `dai-token-tight` (chat reporting; the doc itself is a vault artifact, full prose), `dai-agent-handoff` (shaped these transfer notes), `superpowers:verification-before-completion` (every current-state claim grounded in a file read with path:line; grepped [Authorize] usage and confirmed sports-app has zero bearer/MSAL; ASCII check on new + edited docs). No TDD (docs only, no code change). `dai-grill-me` considered and skipped (the auth direction was already decided in next-platform-architecture-plan.md, not fuzzy). `dai-write-skill` considered and skipped (no new skill). jera-workspace-skills untouched (no approval). Skill-fit note carried forward, unchanged: a dedicated `dai-audit`/`dai-implement-with-vault` skill would fit solo read-and-assess slices better than the interactive grill closing template.

Naming review result (gate item documented): provider named "Microsoft Entra External ID" (the CIAM product, distinct from workforce Entra and from legacy B2C). Provider slugs in `UserIdentity` confirmed consistent with the model comment and next-platform-architecture-plan.md normalization: `google`, `apple`, `microsoft`, `entra` (direct email). Config: keep the existing `AzureAd` section name (conventional even for External ID; renaming churns code for no gain); add `AzureAd__Instance` (`https://{tenant-subdomain}.ciamlogin.com/`) so the authority host is configurable instead of hardcoded `login.microsoftonline.com`; keep `AzureAd__TenantId` and `AzureAd__Audience` (`api://{api-app-id}`). App registrations named to match the resources they back: API app reg `dai-api`, SPA app reg `dai-sports-web`. CORS: deployed `swa-dai-sports-web` origin must be allowed and the `Authorization` header preserved (existing `.AllowAnyHeader()` covers it). Redirect/logout URIs registered on the SPA app registration (not dai-api): local `http://localhost:4201`, cloud `https://{swa-fqdn}`. No misleading names; none of the External ID config values are secrets (SPA public client, no client secret).

### files changed

- `dai-vault/02 Platform/architecture/customer-auth-readiness-v1.md` (new) -- 12-section + future-alignment auth readiness doc: current implementation (verified), External ID decision, four sign-in methods, app registrations/user flow, env vars, local dev behavior, cloud behavior, CORS, redirect/logout URLs, pre-exposure gate, doc reconciliation, docs-only posture, next slice, Azure/AKS/Functions/Postgres alignment. ASCII-clean.
- `dai-vault/02 Platform/architecture/azure-container-apps-provisioning-plan-v1.md` -- section 14 rewritten (External ID confirmed; configurable `AzureAd__Instance` authority; references the readiness doc) and section 18 customer-auth blocker updated to point at the readiness doc gate list.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No `dai` repo changes (docs-only; the narrow authority config correction is deferred to the auth implementation slice, see below). No `jera-workspace-skills` changes.

### auth readiness summary

Real today: .NET JWT Bearer is wired to the workforce authority `login.microsoftonline.com/{tenantId}/v2.0` (`Program.cs:42-53`), the host hardcoded; sports endpoints (`AgentRunsController`, `SportsReferenceController`) carry no `[Authorize]`; `IdentityResolver` hardcodes `provider="entra"`, reads `oid`/`iss`, and has a dev bypass guarded by `env.IsDevelopment() && Dev:EnableBypassAuth`; `UserIdentity` is `(Provider, Issuer, Subject)` with no schema change needed; the sports-app has zero auth (no MSAL, no interceptor, no bearer). Target: Entra External ID as a single JWT issuer federating Google, Apple, Microsoft, and email; the only .NET change is a configurable authority pointed at the `ciamlogin.com` host plus `idp`-claim normalization and JIT provisioning; the only Angular change is MSAL + a bearer interceptor. Local dev is unchanged (dev bypass stays). Cloud requires `Production` + `Dev__EnableBypassAuth=false` and a valid External ID token on every request.

### customer sign-in recommendation

Entra External ID with four methods on one user flow: email one-time passcode (preferred over password to avoid credential storage/reset surface), Google, Apple, and Microsoft (MSA) -- all via External ID federation, no custom OIDC connector needed. Account linking deferred (each provider yields a distinct oid -> distinct UserIdentity row at v1).

### launch blockers (human/portal, before public exposure)

1. Provision the External ID tenant (Azure portal) -- prerequisite for everything else; no validation target exists until it does.
2. Configure Google/Apple/Microsoft/email federations on a sign-up/sign-in user flow.
3. Create the API app registration (`dai-api`: scope `access_as_user`, audience `api://{api-app-id}`) and the SPA app registration (`dai-sports-web`: redirect/logout URIs).
4. Run the auth implementation slice (configurable authority -> External ID; `idp` normalization + JIT in IdentityResolver, drop `tid` splitting; `[Authorize]` on AgentRunsController; MSAL + bearer interceptor in sports-app; environment.ts MSAL config).
5. `Dev__EnableBypassAuth=false` + `ASPNETCORE_ENVIRONMENT=Production` in cloud.
6. CORS allows the deployed web origin and preserves `Authorization`.
7. End-to-end sign-in tested per enabled method before opening the edge.

### future Azure/AKS/Functions/Postgres alignment

External ID changes no topology: `dai-api` is the only public ingress and the token validator; `dai-analyzer` stays internal-only and needs no user token; `swa-dai-sports-web` is the MSAL public client; authority/audience are plain env, not secrets. Future Azure Functions call internal endpoints with a shared-secret header, never a user bearer token. pgvector/Postgres holds no identity (UserIdentity stays in SQL Server). A future AKS move needs no auth change (JWT validation is host-agnostic). Tier enforcement (Stripe) is authorization, separate from this authentication slice.

### next recommended slice

Customer Auth Implementation v1, gated on the External ID tenant existing (blockers 1-3). Order: configurable authority -> idp normalization + JIT -> `[Authorize]` on AgentRunsController -> MSAL + interceptor in sports-app + environment.ts -> externalize CORS + preserve Authorization -> end-to-end sign-in test with bypass off. Last gate before public exposure of `dai-api`. (The Azure Container Apps provisioning execution slice can run in parallel up to but not including public exposure.)

### Claude <-> Codex transfer notes

- Docs-only slice. `dai-vault`: one new doc (`customer-auth-readiness-v1.md`) + provisioning-plan section 14/18 edits + this handoff. `dai` and `jera-workspace-skills` untouched.
- The narrow authority config correction (add `AzureAd:Instance`, build `{Instance}{TenantId}/v2.0`, replace hardcoded `login.microsoftonline.com` in `Program.cs:47`) was deliberately NOT applied: it is only verifiable against a live External ID tenant that does not exist yet, so it lands with the auth slice and is tested in one pass. Do not apply it ahead of the tenant.
- The deeper design narrative (claim mapping table, JIT transaction shape, MSAL interceptor placement) lives in `next-platform-architecture-plan.md` sections 1-2; the readiness doc is the launch-gate summary on top of it.
- Do NOT commit any External ID secret -- there is none for the SPA public client; the API validates tokens with no secret. Tenant id, instance, and audience are non-secret config.
- Pre-existing untracked items in `dai-vault` (20260508/09/10 nba calibration md + artifacts json under `04 Products/sports-v1/calibration/`) are NOT from this slice; leave them.
- No PowerShell or .NET code changed; ASCII check run on the new doc and the edited provisioning plan (both clean).

status: customer auth readiness written 2026-05-22. docs only, no auth code, no Azure resources. provider confirmed Entra External ID federating Google/Apple/Microsoft/email; deploy docs de-ambiguated (section 14/18). manual blockers: External ID tenant + federations + app registrations, then the auth implementation slice. next: Customer Auth Implementation v1. jera-workspace-skills untouched.
