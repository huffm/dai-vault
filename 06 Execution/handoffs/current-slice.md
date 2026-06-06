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

## addendum: Cognitive Protocol Station Blueprint v1 (2026-05-22)

Docs/architecture slice. Defined the station-card contract that turns each cognitive node into a governed "station" with a compact, machine-loadable card, and specified how the Tool Gateway derives permissions from those cards. No runtime code changed (no doc-linked config stub was justified -- the ProtocolRegistry is explicitly a later slice with its own tests). No FastAPI prompt, Pydantic, CognitiveProtocolBuilder, confidence rule, DB schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, secret, or customer-auth change. Cognitive Protocol Runtime behavior, Tool Gateway governance, and CognitiveProtocol persistence preserved.

### naming and skills gate

Skills: `dai-grill-with-vault` (read cognitive-protocol-runtime.md, protocol-node-specs.md, protocol-vocabulary-map.md, current-agent-run-contract.md, and the real Tool Gateway types ToolDefinition/ToolInvocationContext/ToolRegistry before naming or asserting anything), `dai-token-tight` (chat reporting; the doc itself is a vault artifact, full prose), `dai-agent-handoff` (shaped these transfer notes), `superpowers:verification-before-completion` (every facet/field grounded in a file read; confirmed the 7 registered tools, the 3 platform-stage sentinels, and that only AllowedProtocolNodes is enforced in v1; verified both cross-referenced docs exist; ASCII check on the new doc). No TDD (docs only, no code change). `dai-signal-follow-up-diagnostics` considered and skipped (it diagnoses signal coverage on a run, not protocol machinery). `dai-write-skill`/`dai-grill-me` considered and skipped (no new skill; direction already decided in node-specs, not fuzzy). jera-workspace-skills untouched (no approval). Skill-fit note carried forward, unchanged: a dedicated `dai-audit`/`dai-implement-with-vault` skill would fit solo read-and-architect slices better than the interactive grill closing template.

Naming decisions (gate item documented): "protocol station" = the runtime/factory embodiment of a canonical node (node = doctrine word in node-specs; station = factory framing in CLAUDE.md). Load-bearing decision: `station_id` REUSES the existing canonical node id string (e.g. `interrogate.probe`), the same value as `ToolInvocationContext.ProtocolNode` and the tool `AllowedProtocolNodes` entries -- no second id scheme, one keyspace. The 18 station-card fields adopted verbatim from the brief (snake_case), each mapped to a node-specs facet or a `ToolDefinition` field; `cost_class` mirrors `ToolCostClass`, `telemetry_tags` mirror `ToolGatewayInvocation`. Runtime config concept named `ProtocolRegistry v0` (mirrors `ToolRegistry`, static manifest first). Memory tools adopted verbatim, dotted namespace consistent with existing tool ids: `memory.prior_runs.search`, `memory.calibration.lessons`, `memory.niche.rules`, `memory.signal_failures`. Document tools proposed `document.markdown.extract` / `document.html.extract` (governed, bounded, gateway-routed -- not context dumps). Quality gates referenced by existing ids (`counter_case_generic`, `confidence_high_for_partial_evidence`, `frame_missing_rest_context`, ...), not reinvented. Doc filename `protocol-station-blueprint-v1.md` per brief. No misleading names.

### files changed

- `dai-vault/02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md` (new) -- 15-section blueprint: what a station/card is, relation to node-specs, doctrine->config move, the 18 required card fields, gateway permission derivation (node->tools as the dual of tool->nodes), scripts/reflexes, model calls, document tools, vector memory, ML/calibration, sports v1 mapping (NBA/MLB/NFL/NCAAM, NCAAW out of scope), migration path, launch vs post-launch, what not to build. ASCII-clean.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No `dai` repo changes. No `jera-workspace-skills` changes.

### summary

The station card is the machine-readable projection of the node-specs 11 prose facets plus runtime/governance metadata (token_budget, cost_class, telemetry_tags, calibration_hooks, allowed_memory_queries). node-specs stays the source; the card never contradicts it. Tool Gateway permissions become a single matrix readable in both directions: a station may invoke a tool iff the tool is in the card's `allowed_tools` AND the station_id is in the tool's `AllowedProtocolNodes` -- a cross-check, not a second permission system, validated at build time by a future `ProtocolRegistry v0`. Honest current-state caveat captured: the 12 cognitive micro-actions are one analyze call whose gateway caller is `platform.analyze`, so enforcement is at the stage level today; per-station enforcement is honest only after the FastAPI canonical-field migration. Scripts (SignalQualityEvaluator, BuildProbe, SportsComposer, SportsEvaluator) and model calls are both bounded by the card and never widen permissions; Synthesize and the deterministic stations add no facts. Vector memory and document analysis enter only as governed, bounded, gateway-logged tools, never as vault/context dumps -- the same discipline as "load compact station cards."

### risks

- station_id reuses the node id string; if a future ProtocolRegistry diverges from `ToolInvocationContext.ProtocolNode` values, the cross-check (section 6) breaks. Mitigation: the doc names one keyspace explicitly and the validator fails the build on mismatch. Severity: low while docs-only.
- the per-station `allowed_tools` lists are doctrine, not enforced today (one analyze call, stage-level gateway caller). A reader could mistake them for live per-node enforcement. Mitigation: section 6 states the caveat plainly. Severity: low.
- node-specs and the card can drift if hand-maintained. Mitigation: the doc mandates generate-from/validate-against node-specs for ProtocolRegistry v0. Severity: med if the registry slice ignores it.
- NCAAW remains an open gap (out of scope, no overlay). Severity: low; flagged not fabricated.

### next slice

Two reasonable next moves; recommend the registry encoding before any FastAPI rename. (a) `ProtocolRegistry v0`: encode the 15 station cards as a static .NET manifest next to `ToolRegistry`, declarative/unenforced first, plus a startup validator that cross-checks card `allowed_tools` against tool `AllowedProtocolNodes` and fails the build on mismatch (TDD; no behavior change). (b) FastAPI canonical-field migration (the lockstep rename) -- larger, gated on (a) existing so the canonical names have a runtime home. The Azure Container Apps provisioning execution and Customer Auth Implementation v1 slices remain independent and can proceed in parallel.

### Claude <-> Codex transfer notes

- Docs-only slice. `dai-vault`: one new doc (`protocol-station-blueprint-v1.md`) + this handoff. `dai` and `jera-workspace-skills` untouched.
- The new doc is the contract a future `ProtocolRegistry v0` encodes against; it cites the grounding code (`Tools/ToolDefinition.cs`, `ToolInvocationContext.cs`, `ToolRegistry.cs`) and docs (node-specs, vocabulary map, agent-run contract).
- Do NOT add ProtocolRegistry code ahead of its own TDD slice; do NOT split the single analyze call; `allowed_memory_queries` stays empty on every card until pgvector lands offline-first.
- Pre-existing untracked items in `dai-vault` (20260508/09/10 nba calibration md + artifacts json under `04 Products/sports-v1/calibration/`) are NOT from this slice; leave them.
- No PowerShell or .NET code changed; ASCII check run on the new doc (clean).

status: station blueprint written 2026-05-22. docs only, no runtime code, no Azure resources. 18-field station-card contract fixed; gateway permissions derived as the node->tools dual of tool->nodes; memory/document analysis specified as governed tools, not context dumps; migration staged (dual emit -> gateway -> blueprint -> ProtocolRegistry v0 -> FastAPI canonical -> vector memory). next: ProtocolRegistry v0 encoding. jera-workspace-skills untouched.

## addendum: ProtocolRegistry v0 (2026-05-22)

Code slice (TDD). Added the first static .NET protocol station registry: the 15 station cards from the blueprint encoded as a declarative manifest, plus a test-only validator that cross-checks station `allowed_tools` against `ToolRegistry.Default()`. No execution behavior changed: nothing on the run path reads the registry, and the validator is NOT wired into startup. No FastAPI prompt, Pydantic/.NET analysis contract field, CognitiveProtocolBuilder mapping, confidence rule, DB schema, Angular, MCP, pgvector, Azure Functions, or Kubernetes change. Tool Gateway governance, CognitiveProtocol persistence, and dual emit preserved.

### naming and skills gate

Skills: `dai-grill-with-vault` (read the blueprint, node-specs, vocabulary map, agent-run contract, and the real Tool Gateway types ToolDefinition/ToolInvocationContext/ToolRegistry before naming/encoding), `superpowers:test-driven-development` (RED: test project failed to compile on missing `DevCore.Api.Protocols` types -> GREEN after implementing), `superpowers:verification-before-completion` (full suite run; named tests confirmed; ASCII check on the 4 new .cs files), `dai-token-tight` (reporting), `dai-agent-handoff` (these transfer notes). No `dai-write-skill`/`dai-grill-me` (no new skill; design fixed by the blueprint). jera-workspace-skills untouched (no approval). Skill-fit note carried forward: a dedicated `dai-implement-with-vault` skill would fit solo doctrine-to-code slices better than the interactive grill template.

Naming decisions (gate item documented):
- new namespace `DevCore.Api.Protocols` (sibling to `DevCore.Api.Tools`, separate from `AgentRuns` run execution) -- keeps the station registry as doctrine/config, parallel to the tool registry.
- `ProtocolRegistry` with static `Default()` (mirrors `ToolRegistry.Default()`); `ProtocolStationCard` record; `StationIds` constants (mirrors `ToolIds`); `MacroProtocol` enum {Perceive, Interrogate, Discern, Decide, Synthesize}; `StationModelCall` enum {None, SharedAnalyze}; `ProtocolRegistryValidator` + `ProtocolRegistryValidationResult`.
- `cost_class` reuses `ToolCostClass` (Free/Cheap/PaidExternal/ModelCall) -- one cost vocabulary across the tool and protocol manifests, no duplicate enum.
- `station_id` reuses the canonical node id string (e.g. `interrogate.probe`), the same value as `ToolInvocationContext.ProtocolNode` and tool `AllowedProtocolNodes` entries -- one keyspace, no second id scheme (the blueprint's load-bearing decision).
- test names snake_case per the suite convention (`registry_contains_exactly_fifteen_station_cards`, `deliberately_mismatched_card_fails_validation`, ...).

### files changed

- `dai/platform/dotnet/DevCore.Api/Protocols/ProtocolStationCard.cs` (new) -- the v0 card record (10 fields: station id, macro protocol, micro action, purpose, allowed tools, allowed model call, cost class, telemetry tags, quality gates, calibration hooks), `MacroProtocol` + `StationModelCall` enums, `StationIds` constants.
- `dai/platform/dotnet/DevCore.Api/Protocols/ProtocolRegistry.cs` (new) -- `Default()` with all 15 cards. 11 model-emitted stations list `analysis.sports.matchup_read` (SharedAnalyze, ModelCall); 4 deterministic stations (interrogate.probe + synthesize trio) list no tools (None, Free).
- `dai/platform/dotnet/DevCore.Api/Protocols/ProtocolRegistryValidator.cs` (new) -- `Validate(cards, toolRegistry)` returning `ProtocolRegistryValidationResult(IsValid, Errors)`; `ApplicableStageSentinel(card)` derives platform.analyze from SharedAnalyze policy; `DocumentedStageSentinels` constant.
- `dai/platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolRegistryTests.cs` (new) -- 9 tests.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No Program.cs change (validator intentionally not wired into startup). No `jera-workspace-skills` change.

### registry summary

15 cards, one per canonical node. The 11 model-emitted stations (perceive.detect/frame/aim, interrogate.question/verify, discern.weigh/contrast/stress, decide.resolve/position/justify) carry `allowed_tools = [analysis.sports.matchup_read]`, `AllowedModelCall = SharedAnalyze`, `CostClass = ModelCall`: today their content is produced inside the single shared analyze call served by the `platform.analyze` stage, so the analyze tool is the tool whose call produces them. The 4 deterministic stations (interrogate.probe via `BuildProbe`, synthesize.integrate/compose/deliver via `SportsComposer`) carry no tools, `None`, `Free`. Telemetry tags, quality gates, and calibration hooks are doctrine strings grounded in node-specs facet 8 (e.g. decide.position hooks `posture_aligned_with_partial_evidence`/`aggressive_posture_with_partial_evidence`; decide.justify hook `confidence_high_for_partial_evidence`; perceive.frame hooks `frame_missing_spread_context`/`frame_missing_rest_context`). Count nuance documented in code: the canonical "12 cognitive micro-actions" are the perceive/interrogate/discern/decide nodes; of those only Probe is deterministic, so 11 are model-emitted; the synthesize trio (3) is platform-operational. 11 + 4 = 15.

### validation behavior

`ProtocolRegistryValidator.Validate` enforces, per card per listed tool: (1) the tool exists in `ToolRegistry`; (2) the tool's `AllowedProtocolNodes` contains the exact `station_id` OR the station's applicable stage sentinel. The applicable sentinel is derived from the model-call policy: `SharedAnalyze -> platform.analyze`, deterministic `None -> null` (so a deterministic station listing any current tool fails, since no real tool lists a cognitive node id). This is the node->tools dual of the gateway's tool->nodes check; it is honest about today's stage-level enforcement. Test-only: not wired into startup (slice guardrail item 8), so it cannot affect the run path.

### test results

`dotnet test`: 244 passed, 0 failed (was 235, +9). New: `registry_contains_exactly_fifteen_station_cards`, `all_station_ids_are_unique`, `all_required_canonical_station_ids_are_present`, `station_ids_follow_macro_protocol_dot_micro_action_shape`, `every_allowed_tool_id_exists_in_tool_registry`, `default_registry_validates_against_default_tool_registry`, `every_allowed_tool_is_compatible_with_station_id_or_documented_stage_sentinel`, `deliberately_mismatched_card_fails_validation`, `card_listing_unknown_tool_fails_validation`. RED confirmed first (test project compile failure on missing `DevCore.Api.Protocols`). Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:583` is unrelated.

### risks

- the manifest is hand-maintained against node-specs; drift is possible. Mitigation: the validator pins the tool cross-check; a future slice should generate-from or validate-against node-specs. Severity: low (declarative, unread on the run path).
- listing `analysis.sports.matchup_read` on the 11 model stations could read as "this station calls the analyze tool directly." It does not; the platform.analyze stage does. Mitigated by explicit code comments and the sentinel semantics. Severity: low.
- the blueprint v1 prose says "twelve are cognitive (model-emitted)... three are deterministic (Probe plus the Synthesize trio)" -- a minor miscount (Probe + 3 synth = 4 deterministic, 11 model-emitted). The code is precise; the blueprint wording is a doc nit to fix in a later touch (not edited this slice, out of scope). Severity: low.
- validator not wired to startup, so a future bad edit to either manifest is caught only when tests run. Acceptable for v0; promotion to a startup guard is a later, deliberate step. Severity: low.

### next slice

Two options. (a) Promote the validator to a startup guard (fail fast on a manifest/tool mismatch at boot) once it has run green in CI -- small, but changes startup behavior, so gate it behind tests and a deliberate decision. (b) Begin the FastAPI canonical-field migration (the lockstep rename) so per-station model construction and per-station gateway enforcement become honest, at which point `allowed_tools` stops needing the stage-sentinel stand-in. Recommend (b) is the larger arc but (a) is the cheaper immediate hardening; do (a) only if startup-failure-on-mismatch is wanted now. Provisioning execution and Customer Auth Implementation v1 remain independent.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (4 new files under `DevCore.Api/Protocols/` and `DevCore.Api.Tests/Protocols/`) and `dai-vault` (this handoff). `jera-workspace-skills` read-only, untouched.
- Re-verify anywhere: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 244 passing. No stack/DB/network needed (the registry and validator are pure static data + computation).
- Extension pattern: add a `StationIds` constant + a `ProtocolStationCard` in `ProtocolRegistry.Default()`; if a station gains a real per-station tool call later, set its `allowed_tools` and the validator checks it by exact station id once the tool's `AllowedProtocolNodes` is opened to that node (dropping the stage-sentinel stand-in).
- The validator is intentionally not in `Program.cs`. Do not wire it in without a deliberate decision; if promoted, add a startup test first.
- Pre-existing untracked `dai-vault` calibration files under `04 Products/sports-v1/calibration/` are NOT from this slice; leave them.
- No PowerShell changed; ASCII verified on the 4 new .cs files (comments are lowercase/ascii per CLAUDE.md).

status: ProtocolRegistry v0 merged 2026-05-22. 15 station cards encoded in DevCore.Api/Protocols; test-only validator cross-checks allowed_tools against ToolRegistry via the platform.analyze stage sentinel. 244 tests passing (was 235, +9). no run-path behavior change, validator not wired to startup. next: validator-as-startup-guard (optional) or FastAPI canonical-field migration. jera-workspace-skills untouched.

## addendum: Protocol Registry Startup Guard v1 (2026-05-22)

Code slice (TDD). Promoted the ProtocolRegistry v0 validator from test-only to a startup guard: Program.cs now validates `ProtocolRegistry.Default()` against `ToolRegistry.Default()` during host configuration and fails fast with a clear `InvalidOperationException` on drift. No run-path behavior change when validation passes. No FastAPI prompt, Pydantic/.NET analysis contract field, CognitiveProtocolBuilder mapping, confidence rule, DB schema, Angular, MCP, pgvector, Azure Functions, or Kubernetes change. Tool Gateway governance, CognitiveProtocol persistence, and dual emit preserved.

### naming and skills gate

Skills: `dai-grill-with-vault` (read the existing validator, the `AddDaiToolGateway` extension pattern, the `ToolGatewayDIRegistrationTests` minimal-factory pattern, and the Program.cs wire-in point before naming/coding), `superpowers:test-driven-development` (RED: test project failed to compile on the missing `ProtocolRegistryServiceCollectionExtensions` -> GREEN after implementing), `superpowers:verification-before-completion` (full suite run; 4 new tests confirmed by name; ASCII check on the new/edited .cs files), `dai-token-tight` (reporting), `dai-agent-handoff` (these notes). No new skill (`dai-write-skill`/`dai-grill-me` not applicable). jera-workspace-skills untouched (no approval). Skill-fit note carried forward: a dedicated `dai-implement-with-vault` skill would fit these solo doctrine-to-code slices better than the interactive grill template.

Naming decisions (gate item documented):
- new static class `ProtocolRegistryServiceCollectionExtensions` in `DevCore.Api.Protocols` -- mirrors `ToolGatewayServiceCollectionExtensions` in `DevCore.Api.Tools`.
- entry method `ValidateDaiProtocolRegistry(this IServiceCollection)` -- `Validate` verb (it validates, it does not register services) with the `Dai` prefix matching the `AddDaiToolGateway` family; returns the collection for chaining.
- testable throw core `ThrowIfProtocolRegistryInvalid(IReadOnlyList<ProtocolStationCard>, IToolRegistry)` -- public so tests drive the throw path with a synthetic bad manifest without needing InternalsVisibleTo.
- failure type `InvalidOperationException` (per brief); message prefixed "Protocol registry validation failed against the tool registry:" then one line per validator error (each already names station id + tool id).
- test names snake_case per suite convention (`application_startup_succeeds_when_registries_agree`, `invalid_registry_fails_the_guard_with_invalid_operation_exception`, `guard_error_message_includes_station_id_and_tool_id`).

### files changed

- `dai/platform/dotnet/DevCore.Api/Protocols/ProtocolRegistryServiceCollectionExtensions.cs` (new) -- `ValidateDaiProtocolRegistry` extension + `ThrowIfProtocolRegistryInvalid` core. Validates static metadata only; no db/cloud/http dependency; no node-spec markdown read.
- `dai/platform/dotnet/DevCore.Api/Program.cs` -- added `using DevCore.Api.Protocols;` and one line `builder.Services.ValidateDaiProtocolRegistry();` immediately after `AddDaiToolGateway()`, before `builder.Build()`.
- `dai/platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolRegistryStartupGuardTests.cs` (new) -- 4 tests: WebApplicationFactory startup-succeeds smoke, default registry passes the guard, mismatched card throws InvalidOperationException, error message includes station id + tool id.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No change to `ProtocolRegistry.cs`, `ProtocolStationCard.cs`, `ProtocolRegistryValidator.cs`, the Tool Gateway, SportsRetriever/Analyzer/Composer, or FastAPI. No `jera-workspace-skills` change.

### behavior summary

At host configuration time Program.cs calls `ValidateDaiProtocolRegistry()`, which runs the existing `ProtocolRegistryValidator` over `ProtocolRegistry.Default()` vs `ToolRegistry.Default()`. The manifests agree today, so startup is unchanged and no request path is touched. If a future edit makes a station list a tool the tool registry does not grant it (by exact station id or the applicable stage sentinel) or a nonexistent tool, the host throws `InvalidOperationException` before `builder.Build()` returns -- the app does not start, and the message lists each offending station id + tool id. The guard is synchronous, pure-metadata, and dependency-free, so it adds no startup I/O.

### test results

`dotnet test`: 248 passed, 0 failed (was 244, +4). New: `application_startup_succeeds_when_registries_agree` (real Program.cs host builds via WebApplicationFactory with the guard wired in), `default_protocol_registry_passes_the_startup_guard`, `invalid_registry_fails_the_guard_with_invalid_operation_exception`, `guard_error_message_includes_station_id_and_tool_id`. RED confirmed first (compile failure on missing `ProtocolRegistryServiceCollectionExtensions`). The 9 ProtocolRegistry v0 tests still pass. Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:583` is unrelated.

### risks

- the guard now gates app startup. A genuine manifest/tool mismatch will stop the app from booting. This is the intended fail-fast behavior, and it can only fire on a developer edit to one of the two static manifests (not on runtime data), caught immediately in dev/CI. Severity: low (desired hardening; the WebApplicationFactory test proves the current manifests boot clean).
- validation runs on every host start (including each WebApplicationFactory in the suite). Cost is a single pass over 15 cards x their tool lists -- negligible. Severity: low.
- `ThrowIfProtocolRegistryInvalid` is public for testability; it is a thin throwing wrapper over the validator, not new permission logic. Severity: low.

### next slice

The protocol manifest is now self-checking at boot. The larger forward arc is the FastAPI canonical-field migration (the lockstep rename of the analyze prompt + Pydantic + .NET records) so per-station model construction and per-station gateway enforcement become honest, at which point station `allowed_tools` validates by exact station id and the stage-sentinel stand-in retires. Azure Container Apps provisioning execution and Customer Auth Implementation v1 remain independent and gated only on their own manual blockers.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (1 new extension, 1 new test, Program.cs +2 lines) and `dai-vault` (this handoff). `jera-workspace-skills` read-only, untouched.
- Re-verify anywhere: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 248 passing. No stack/DB/network needed (in-memory EF + stubbed identity in the factory; the guard is pure static metadata).
- The guard validates the static `*.Default()` manifests, not the DI-registered `IToolRegistry`; that is intentional and dependency-free. If a future slice makes the tool manifest dynamic, revisit whether the guard should read the registered registry instead.
- To trip the guard deliberately (sanity check): add a `ProtocolStationCard` whose `AllowedTools` lists a tool whose `AllowedProtocolNodes` excludes both the station id and its applicable sentinel; the app will refuse to start with a message naming that station and tool.
- Pre-existing untracked `dai-vault` calibration files under `04 Products/sports-v1/calibration/` are NOT from this slice; leave them.
- No PowerShell changed; ASCII verified on the new/edited .cs files.

status: startup guard merged 2026-05-22. Program.cs validates ProtocolRegistry.Default() vs ToolRegistry.Default() at host configuration and throws InvalidOperationException on drift; passing leaves runtime unchanged. 248 tests passing (was 244, +4). next: FastAPI canonical-field migration. jera-workspace-skills untouched.

## addendum: FastAPI Canonical Field Migration Plan v1 (2026-05-22)

Planning slice (docs only). Produced the lockstep plan to rename the analyzer's legacy cognitive-phase fields to canonical protocol fields across the FastAPI prompt, the Pydantic models, and the .NET wire contract. No runtime code changed. No FastAPI prompt, Pydantic model, .NET contract, CognitiveProtocolBuilder mapping, confidence rule, DB schema, Angular, Tool Gateway, MCP, pgvector, Azure Functions, or Kubernetes change.

### naming and skills gate

Skills: `dai-grill-with-vault` (read sports_analyzer.py prompt + `_parse_phases`, app/models/sports.py, SportAnalysisContracts.cs, CognitiveProtocolBuilder.cs, and the vocabulary map before planning), `dai-token-tight` (chat reporting; the doc is full prose), `dai-agent-handoff` (these notes), `superpowers:verification-before-completion` (every field claim grounded in a file read with line refs; ASCII check on the new doc), `superpowers:writing-plans`/`planning` (sequenced, revertible implementation steps with explicit gates). No TDD (no code). jera-workspace-skills untouched (no approval). Skill-fit note carried forward: a dedicated `dai-implement-with-vault` (or a `dai-migration-plan`) skill would fit lockstep cross-language migration planning better than the interactive grill template.

Naming review result (gate item documented): canonical field names are LOCKED by the vocabulary map and station ids -- reused exactly (detect/frame/aim, question/probe/verify, weigh/contrast/stress, resolve/position/justify); no new vocabulary invented. Pydantic + .NET class renames specified to the canonical `...Protocol` suffix (SportsCognitivePhases -> SportsCognitiveProtocol, etc.) so both sides share one vocabulary; response member `phases -> protocol` (flagged as Q5). Compatibility aliases use Pydantic `validation_alias`/`AliasChoices` and .NET `[JsonPropertyName]` for a transition window. Delivery field names (`counter_case`, `watch_for`, `what_would_change_the_read`, `lean`, `lean_side`) are NOT renamed -- they are product extracts, not protocol fields. New artifact version constant `sports_decision_artifact_v3` proposed (parallel to v2). Migration steps named and ordered (aliases -> renames -> class/member rename -> structural Stress -> array/scalar -> v3 stamp -> alias removal -> doctrine update).

### files changed

- `dai-vault/02 Platform/architecture/cognitive-factory/fastapi-canonical-field-migration-plan-v1.md` (new) -- 15-section plan: legacy map, canonical map, Pydantic changes, prompt changes, .NET contract changes, persisted-field compatibility, dual-emit decision, builder role, /dev/artifacts handling, calibration handling, test plan, rollback, exact sequence, do-not-change list, open questions. ASCII-clean.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No `dai` repo changes. No `jera-workspace-skills` changes.

### migration plan summary

The migration is NOT a flat rename. Eight pure renames (balance->question, reframe->verify, filter->weigh, listen->contrast, voice->resolve, calibrate->justify, plus frame and posture->position keeping value semantics) are low risk. Two changes are structural and approval-gated: (1) Stress relocates Interrogate->Discern AND collapses two legacy stress-adjacent prompts (interrogate.stress + discern.test) into one canonical discern.stress -- a prompt-content change with calibration impact; (2) perceive.detect/aim are arrays today but scalars in the canonical CognitiveProtocol record. Probe (deterministic BuildProbe) and Synthesize (deterministic constants) are never model-emitted, so no prompt change for them. Recommended approach: no model-side dual-emit (divergence + token risk); instead an alias window at the parser boundary decouples prompt and contract rollouts and gives clean rollback. CognitiveProtocolBuilder becomes pass-through for the renamed scalars while retaining Probe, Synthesize, array-join, and Stress sourcing. New records stamp `sports_decision_artifact_v3`; old v1/v2 records are never rewritten and continue to project via ProtocolView. /dev/artifacts and calibration scripts already prefer canonical CognitiveProtocol, so they need confirmation, not redesign.

### risks

- the Stress collapse (Q1) changes model output and may shift calibration; mitigated by landing it as its own revertible commit with a calibration spot-check. Severity: med.
- lockstep drift between FastAPI and .NET if the prompt and contract roll out separately without the alias window; mitigated by shipping aliases first (step 1). Severity: med without aliases, low with.
- array->scalar (Q2) is a wire-type change; recommend keeping arrays to avoid prompt churn. Severity: low.
- dropping legacy CognitivePhases on v3 (Q3) could surprise a consumer that reads the legacy block directly; mitigated because the read-side projection and /dev/artifacts already prefer canonical. Severity: low.
- this is a plan; the risk is mostly deferred to the implementation slice, which must hold the do-not-change list (section 14) and get approval on the six open questions before coding.

### recommended implementation slice

FastAPI Canonical Field Migration v1 (code), gated on approval of the six open questions (especially Q1 Stress collapse and Q4 alias window). Land in the section-13 order: (1) add aliases additive, (2) pure renames in prompt+parser+builder+tests, (3) class + `phases->protocol` rename, (4) structural Stress as an isolated commit with a calibration spot-check, (5) array/scalar per Q2, (6) stamp v3 + decide legacy-block persistence, (7) remove aliases after a calibration window, (8) update the vocabulary map + node-specs to mark legacy names retired-from-runtime. TDD throughout; the 248-test suite plus pytest stay green; confidence rules and posture enum untouched.

### Claude <-> Codex transfer notes

- Planning slice. `dai-vault`: one new doc (`fastapi-canonical-field-migration-plan-v1.md`) + this handoff. `dai` and `jera-workspace-skills` untouched.
- The plan is the build spec for the migration code slice; start at section 13 (exact sequence) and clear section 15 (open questions) with the human first.
- Key grounding: the legacy->canonical remap currently lives in `CognitiveProtocolBuilder.FromLegacy` (DevCore.Api/AgentRuns) and the prompt JSON in `sports_analyzer.py` lines 58-80 + 118-168; the .NET wire mirror is `SportAnalysisContracts.cs`. The Stress relocation (interrogate.stress|discern.test -> discern.stress) is the crux.
- Do NOT start coding without Q1/Q4 approval; do NOT make the model emit two shapes (no model dual-emit); do NOT rewrite historical records.
- Pre-existing untracked `dai-vault` calibration files under `04 Products/sports-v1/calibration/` are NOT from this slice; leave them.
- No code or PowerShell changed; ASCII check run on the new doc (clean).

status: migration plan written 2026-05-22. docs only, no runtime code. eight pure renames + two structural changes (Stress relocation/collapse, detect/aim array->scalar) identified; alias-window lockstep approach with no model dual-emit; CognitiveProtocolBuilder narrows to pass-through + deterministic Probe/Synthesize; v3 artifact version proposed; six open questions need approval before coding. next: FastAPI Canonical Field Migration v1 (code, gated on approvals). jera-workspace-skills untouched.

## addendum: FastAPI Canonical Alias Scaffold v1 (2026-05-22)

Code slice (TDD, Python + .NET). Step 1 of the migration plan: make both the FastAPI parser/models and the .NET client contract ACCEPT canonical cognitive-field names while still accepting legacy names, without changing any emitted/serialized output. No prompt change, no persisted-field rename, no v3 stamp, no Stress collapse. No confidence rule, DB schema, Angular, Tool Gateway, MCP, pgvector, Azure Functions, or Kubernetes change. Current runtime behavior preserved (legacy is what is emitted today and legacy keeps precedence on input).

### naming and skills gate

Skills: `dai-grill-with-vault` (read the analyzer prompt + `_parse_phases`/`_parse_response`, sports.py models, SportAnalysisContracts.cs, FastApiClient.cs, and the migration plan before coding -- discovered the analyzer hand-parses model JSON via dict.get, which determined the real alias seam), `superpowers:test-driven-development` (RED on both sides: 3 failing pytest canonical tests, then .NET compile failure on the missing wire type -> GREEN), `superpowers:verification-before-completion` (full pytest + full dotnet test, named tests confirmed, ASCII check on all changed files), `dai-token-tight` (reporting), `dai-agent-handoff` (these notes). jera-workspace-skills untouched (no approval). Skill-fit note carried forward: a `dai-implement-with-vault`/`dai-migration` skill suited to cross-language lockstep edits would beat the interactive grill template here.

Naming decisions (gate item documented):
- canonical field names reused exactly from the vocabulary map (question/verify/contrast/weigh/justify/resolve/position); `protocol` is the canonical block-key alias for `phases`. No new vocabulary.
- FastAPI: aliases realized in TWO places -- (a) the hand-parser `_parse_phases`/`_parse_response` via a `_pick(d, *keys)` legacy-first helper (the runtime-meaningful seam, since the analyzer builds models positionally), and (b) Pydantic `validation_alias=AliasChoices(legacy, canonical)` + `model_config = ConfigDict(populate_by_name=True)` on the renamed fields (satisfies "in the models" and enables model_validate; populate_by_name keeps the positional constructor working; no serialization_alias so output stays legacy).
- .NET: a tolerant `SportsAnalysisResponseWire` (+ `CognitivePhasesWire` and the four phase wires) with dual `[JsonPropertyName]` properties per renamed field and both `phases`/`protocol`, mapping via `ToResponse()`/`ToModel()` (legacy ?? canonical) into the UNCHANGED `SportsAnalysisResponse`/`SportsCognitivePhases`. STJ has no native multi-alias, so a wire DTO is the idiomatic, low-risk choice (vs a converter).
- Stress (interrogate.stress) and the discern.test source are deliberately NOT aliased on either side -- the Stress relocation/collapse is the next, isolated slice (plan Q1).

### files changed

- `dai/services/agent-service/app/models/sports.py` -- `validation_alias` + `populate_by_name` on `SportsInterrogatePhase` (balance|question, reframe|verify), `SportsDiscernPhase` (listen|contrast, filter|weigh), `SportsDecidePhase` (calibrate|justify, posture|position, voice|resolve), and `SportsAnalysisResponse.phases` (phases|protocol). Import line gains `AliasChoices, ConfigDict`. Serialization unchanged.
- `dai/services/agent-service/app/services/sports_analyzer.py` -- `_pick` helper in `_parse_phases`; canonical-or-legacy reads for the renamed nested fields and the nested posture/position; `_parse_response` accepts `phases` or `protocol` block. Stress/test reads unchanged.
- `dai/services/agent-service/tests/test_sports_analyzer.py` -- 7 new tests (legacy still parses, canonical parses, protocol-key alias, legacy precedence, model_validate canonical, model_validate legacy) + import of the three phase models.
- `dai/platform/dotnet/DevCore.AiClient/SportsAnalysisWire.cs` (new) -- the tolerant wire DTO + mappers.
- `dai/platform/dotnet/DevCore.AiClient/FastApiClient.cs` -- deserialize `SportsAnalysisResponseWire` then `ToResponse()` (was binding `SportsAnalysisResponse` directly).
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsAnalysisWireTests.cs` (new) -- 5 tests (legacy maps unchanged, canonical maps equivalent, protocol alias, legacy precedence, null phases).
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No change to the prompt JSON output, `CognitiveProtocol.cs`, `CognitiveProtocolBuilder.cs`, `AgentRunContracts.cs`, `SportsAnalyzer.cs`, the ToolGateway, or any persisted field. `jera-workspace-skills` untouched.

### alias scaffold summary

The analyzer still emits legacy field names; nothing user-facing changes. The FastAPI parser and Pydantic models now ALSO accept canonical names (and the `protocol` block key), legacy-first. On the .NET side, `FastApiClient` reads a tolerant wire DTO that accepts both name sets and maps to the unchanged `SportsAnalysisResponse`; for today's legacy payload the mapping is effect-identical to the prior direct deserialization. This decouples the future prompt flip from a .NET deploy: when a later slice flips the prompt to canonical names, both services already parse it. Stress relocation/collapse and the v3 stamp remain deferred.

### test results

- pytest (`tests/test_sports_analyzer.py`): 87 passed (was 80; +7). RED first on the 3 canonical/protocol tests, GREEN after the parser + model aliases.
- dotnet test (full suite): 253 passed, 0 failed (was 248; +5 wire tests). RED first (compile failure on missing `SportsAnalysisResponseWire`), GREEN after the wire DTO + FastApiClient wire-in. The pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:583` is unrelated.

### risks

- `FastApiClient` now maps through the wire DTO; for legacy input it must produce exactly the prior response. Covered by `legacy_payload_maps_to_response_unchanged` and the existing analyzer/integration tests; `Summary!`/`Factors!` are passed through to preserve the prior (non-validated) behavior. Severity: low.
- pydantic `populate_by_name=True` is accepted in 2.12.5 (installed) but is the older spelling (newer `validate_by_name`/`validate_by_alias`); a future pydantic major could deprecate it. Severity: low; pinned by the unchanged requirement and covered by tests.
- the wire DTO duplicates the response shape; if `SportsAnalysisResponse` gains a field, the wire DTO must too. Documented; a parity test could be added later. Severity: low.
- validation_alias on the models is partly redundant with the parser (the runtime path hand-parses), but it satisfies the literal "in the models" requirement and the model_validate tests. Severity: none (additive).

### next recommended slice

Plan step 2-3: the pure renames in the prompt + the `phases->protocol`/class renames, now safe because both sides already accept canonical (this scaffold). Then the gated structural slices: Stress collapse (Q1, isolated commit + calibration spot-check) and detect/aim array-vs-scalar (Q2), then v3 stamp (Q6) + legacy-block persistence decision (Q3), then alias removal (step 7). Confidence rules and posture enum stay untouched throughout.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (FastAPI: 3 files; .NET: 1 new wire file, FastApiClient edit, 1 new test) and `dai-vault` (this handoff). `jera-workspace-skills` untouched.
- Re-verify: FastAPI `\.venv\Scripts\python -m pytest tests/test_sports_analyzer.py` -> 87 passing; .NET `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 253 passing. No stack/DB/network needed.
- The real FastAPI alias seam is the parser (`_pick`), because the analyzer hand-parses model JSON; `validation_alias` on the models is additive and only affects `model_validate`. Keep both.
- The .NET seam is the wire DTO in `DevCore.AiClient/SportsAnalysisWire.cs`; `FastApiClient` maps through it. Production traffic stays legacy until the prompt flips, so the canonical path is exercised only by tests this slice.
- Do NOT flip the prompt, collapse Stress, stamp v3, or rename persisted fields in this scaffold -- those are later plan steps. legacy keeps precedence everywhere.
- Pre-existing untracked `dai-vault` calibration files are NOT from this slice; leave them.

status: alias scaffold merged 2026-05-22. FastAPI parser + Pydantic models and the .NET wire DTO accept canonical (and `protocol`) names while still accepting legacy; legacy emitted + legacy-precedence so runtime behavior is unchanged. pytest 87 passing (+7), dotnet 253 passing (+5). Stress collapse / v3 / prompt flip deferred. next: pure renames + phases->protocol rename. jera-workspace-skills untouched.

## addendum: Canonical Prompt Pure Rename v1 (2026-05-22)

Code slice (TDD, FastAPI prompt only). Flipped the analyzer prompt to request the canonical names for the seven approved pure-rename fields. Backward-compatible because the parser still accepts legacy via the alias scaffold. No structural Stress collapse, no v3 stamp, no alias removal, no Probe/Synthesize change, no confidence/schema/Angular/Tool Gateway/MCP/pgvector/Functions/Kubernetes change. The persisted artifact shape and `CognitiveProtocolBuilder` are untouched (the record fields stay legacy-named; the canonical prompt keys are alias-mapped to them).

### naming and skills gate

Skills: `dai-grill-with-vault` (read the full prompt `_JSON_SHAPE`, `_parse_phases`/`_parse_response`, the scaffold tests, and the migration plan before editing), `superpowers:test-driven-development` (RED on prompt-content tests -> GREEN), `superpowers:systematic-debugging` (caught a self-inflicted regression: a too-broad `replace_all` rewrote Python attribute access `phases.decide.posture`/`phases.interrogate.balance` in `_parse_response`, breaking 20 parse tests; reverted those two code lines to the legacy record-field names, comments updated), `superpowers:verification-before-completion` (full pytest + full dotnet + a prompt-region grep confirming no renamed legacy label remains and stress/test stay), `dai-token-tight`, `dai-agent-handoff`. jera-workspace-skills untouched (no approval). Skill-fit note carried forward, plus a sharpening note: a `dai-implement-with-vault` skill could include a "rename field labels in prompt strings but never in attribute-access code" guard -- the replace_all footgun this slice hit.

Naming review result (against vocabulary map + station registry): the seven canonical names match the registry station ids exactly -- `interrogate.question`, `interrogate.verify`, `discern.weigh`, `discern.contrast`, `decide.resolve`, `decide.position`, `decide.justify`. Top-level delivery fields (`posture`, `counter_case`, `watch_for`) keep their names; only their nested cross-references in the prompt moved to canonical. `interrogate.stress` and `discern.test` stay legacy (collapse deferred). No new vocabulary.

### files changed

- `dai/services/agent-service/app/services/sports_analyzer.py` -- `_JSON_SHAPE` only: skeleton keys and per-field instruction labels flipped for the seven pure renames; the cross-reference lines updated (`posture must match phases.decide.position`, `counter_case must ... phases.interrogate.question`); `watch_for ... phases.interrogate.stress` left legacy. Two `_parse_response` lines that a broad replace_all wrongly touched were reverted to the legacy record-field attribute access (`phases.decide.posture`, `phases.interrogate.balance`) with clarifying comments. `_parse_phases` (the alias scaffold `_pick` reads) is unchanged.
- `dai/services/agent-service/tests/test_sports_analyzer.py` -- 4 new tests: prompt emits canonical labels, prompt no longer asks for legacy labels (dotted-label absence + quoted-skeleton-key absence), stress/test stay legacy, and a canonical-prompt-shaped payload parses through `_parse_response` into the legacy-named record fields.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No .NET change this slice (the wire DTO from the scaffold already accepts canonical; dotnet suite re-run green as a regression guard). No prompt change beyond the seven renames. `jera-workspace-skills` untouched.

### behavior summary

The analyzer now asks the model for canonical field names on the seven pure-rename fields; the model's canonical output is read by the parser's alias scaffold (`_pick` legacy-or-canonical) and mapped into the unchanged legacy-named `SportsCognitivePhases` record, so `CognitiveProtocolBuilder`, the persisted `OutputJson` shape, the v2 stamp, and all downstream consumers are byte-unaffected. Legacy payloads still parse (alias scaffold + tests). The top-level delivery extracts and the deterministic Probe/Synthesize are untouched. Stress and test remain legacy pending their own slice.

### test results

- pytest: 91 passed (was 87; +4). RED first on the two prompt-content tests; one extra RED cycle from the replace_all regression (20 parse failures) then GREEN after reverting the two attribute-access lines and tightening the legacy-key test to quoted-key form (avoids a false positive on the word "calibrated").
- dotnet test (full suite): 253 passed, 0 failed (unchanged; no .NET code touched). Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:583` unrelated.
- verification item 4 confirmed by grep: no renamed legacy field label remains in the `_JSON_SHAPE` region; `phases.interrogate.stress` and `phases.discern.test` still present.

### risks

- prompt wording change can shift model output phrasing/quality even for a pure rename; mitigated because only field labels changed (the instruction prose and prohibited-phrase guardrails are identical), and confidence/posture remain deterministic platform-side. A calibration spot-check on the next live batch is prudent but not required (no scoring logic changed). Severity: low-med.
- the legacy record field names now diverge from the canonical prompt keys (intentional, alias-mapped). A future reader could be confused that the prompt says `question` but the record says `Balance`; documented in the reverted-line comments and here. Severity: low.
- replace_all on dotted strings is a footgun when the same string is both a prompt label and an attribute access; caught and fixed this slice. Severity: resolved.

### next recommended slice

Plan step 3 continued: the class/member rename (`SportsCognitivePhases` -> `SportsCognitiveProtocol`, `phases` -> `protocol`) in lockstep across Pydantic + .NET, now that the prompt emits canonical and both sides accept it. Then the gated structural slices: Stress collapse (Q1, isolated commit + calibration spot-check), detect/aim array-vs-scalar (Q2), v3 stamp (Q6) + legacy-block persistence (Q3), then alias removal (step 7). Confidence rules and posture enum stay untouched.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (FastAPI: `sports_analyzer.py` prompt + 2 reverted `_parse_response` lines, plus test additions) and `dai-vault` (this handoff). `jera-workspace-skills` untouched.
- Re-verify: FastAPI `\.venv\Scripts\python -m pytest tests/test_sports_analyzer.py` -> 91 passing; .NET `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 253 passing.
- GOTCHA for the next renamer: in `sports_analyzer.py` the strings `phases.decide.posture` and `phases.interrogate.balance` appear BOTH as prompt instruction labels (rename) AND as Python attribute access in `_parse_response` reading the legacy-named record (do NOT rename until the record fields are renamed). Do not blanket replace_all.
- The record fields are still legacy-named (`Balance`, `Reframe`, `Listen`, `Filter`, `Calibrate`, `Voice`, `Posture`); the class/member rename is the next slice. Until then the canonical prompt keys are alias-mapped to legacy record fields by `_pick`.
- Do NOT collapse stress, stamp v3, remove aliases, or touch the .NET wire DTO this far. Legacy still accepted everywhere.
- Pre-existing untracked `dai-vault` calibration files are NOT from this slice; leave them.

status: canonical prompt pure rename merged 2026-05-22. analyzer prompt now emits canonical names for the seven pure-rename fields; parser alias-maps them to the unchanged legacy record fields, so persisted shape and CognitiveProtocolBuilder are unaffected. pytest 91 (+4), dotnet 253 (unchanged). stress/test still legacy; v3/collapse/alias-removal deferred. next: class/member rename (phases->protocol). jera-workspace-skills untouched.

## addendum: Analyzer Protocol Class and Member Rename v1 (2026-05-26)

Code slice (TDD, Python + .NET). Renamed the analyzer response wrapper from legacy "phases" naming to canonical "protocol" naming, with full backward compatibility. No structural Stress collapse, no v3 stamp, no alias removal, no Probe/Synthesize change, no confidence/schema/Angular/Tool Gateway/MCP/pgvector/Functions/Kubernetes change. Persisted artifact behavior unchanged.

### naming and skills gate

Skills: `dai-grill-with-vault` (read the Pydantic models, `_parse_response`, the route's `response_model` serialization path, the .NET wire DTO, and traced `SportsCognitivePhases` usage across 10 .NET files before scoping), `superpowers:test-driven-development` (RED on 6 pytest + 1 .NET test -> GREEN), `superpowers:verification-before-completion` (full pytest + full dotnet + ASCII check; confirmed the 44 existing `.phases` reads stay green via the property), `dai-token-tight`, `dai-agent-handoff`. jera-workspace-skills untouched (no approval). Skill-fit note carried forward: a `dai-implement-with-vault` skill should encode the "rename at the contract surface, not the deeply-embedded downstream type" scoping heuristic used here.

Naming decisions (gate item documented):
- Pydantic: `SportsCognitivePhases` -> `SportsCognitiveProtocol`, with a module-level back-compat alias `SportsCognitivePhases = SportsCognitiveProtocol` (importers do not break). Sub-phase classes (`SportsPerceivePhase`, `SportsInterrogatePhase`, `SportsDiscernPhase`, `SportsDecidePhase`) intentionally NOT renamed -- only the aggregate and the response member were approved.
- Pydantic response member: `phases` -> `protocol` (canonical), `validation_alias=AliasChoices("protocol","phases")` (legacy `phases` key still accepted), serialization now emits `protocol`; a read-only `@property phases` returns `self.protocol` so the 44 existing `.phases` accessors (and any consumer) keep working without churn.
- .NET wire DTO: `CognitivePhasesWire` -> `CognitiveProtocolWire`; the wire response member `Protocol` is now primary (first, `[JsonPropertyName("protocol")]`) and `Phases` is the retained legacy alias; `ToResponse()` prefers `Protocol ?? Phases`.
- SCOPING DECISION (documented + deferred): the .NET downstream contract record `SportsCognitivePhases` (in `SportAnalysisContracts.cs`), `SportsAnalysisResponse.Phases`, `CognitiveProtocolBuilder`, `ProtocolVocabularyMapper`, and the persisted-block type are intentionally LEFT UNCHANGED. Renaming that record ripples into 10 files including the legacy->canonical builder and the persisted contract, with no behavior benefit this slice; item 6 mandates mapping into existing downstream contracts without changing them. The .NET wire DTO renames; the downstream contract rename is a separate later slice.

### files changed

- `dai/services/agent-service/app/models/sports.py` -- class rename + back-compat alias; response member `phases`->`protocol` with validation alias + `phases` read-property; comment fixes.
- `dai/services/agent-service/app/services/sports_analyzer.py` -- import `SportsCognitiveProtocol`; `_parse_phases` return type + constructor; `_parse_response` constructs `SportsAnalysisResponse(protocol=...)`.
- `dai/services/agent-service/tests/test_sports_analyzer.py` -- 6 new tests (protocol member, phases-property alias, serializes protocol-not-phases, model_validate protocol + phases-alias, back-compat class alias).
- `dai/platform/dotnet/DevCore.AiClient/SportsAnalysisWire.cs` -- `CognitivePhasesWire`->`CognitiveProtocolWire`; `Protocol` primary + `Phases` alias; `ToResponse()` prefers protocol.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsAnalysisWireTests.cs` -- 1 new test (protocol preferred over phases when both present).
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No change to FastApiClient, the .NET contract record `SportsCognitivePhases`, `CognitiveProtocolBuilder`, `ProtocolVocabularyMapper`, or any persisted field. `jera-workspace-skills` untouched.

### behavior summary

The analyzer response's cognitive block is now the canonical `protocol` member end to end at the analyzer-wire surface: FastAPI serializes `protocol`; the .NET wire DTO reads `protocol` (preferred) or `phases` (legacy alias) and maps into the unchanged `SportsCognitivePhases` .NET contract record, so `CognitiveProtocolBuilder`, the persisted `OutputJson` shape, and the v2 stamp are byte-unaffected. The deterministic Probe/Synthesize, the posture enum, and confidence are untouched. Stress/test stay legacy.

### compatibility behavior

- FastAPI: `protocol` member is canonical; `phases` accepted as a validation alias on input; `.phases` property preserves legacy read access; `SportsCognitivePhases` name still importable (alias). Legacy and canonical payloads both parse; serialization emits `protocol`.
- .NET: wire accepts both `protocol` (primary) and `phases` (alias); downstream contract `SportsAnalysisResponse.Phases` unchanged, so all consumers and persisted artifacts are unaffected. Old persisted OutputJson records are never touched and remain readable.

### test results

- pytest: 97 passed (was 91; +6). RED first on the 6 rename tests. The 44 pre-existing `.phases` reads pass unchanged via the property.
- dotnet test (full suite): 254 passed, 0 failed (was 253; +1 protocol-preference test). RED first on that test (Phases won before the precedence flip). Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:583` unrelated.

### risks

- FastAPI now serializes `protocol` instead of `phases` to .NET. Safe because the .NET wire DTO accepts both and prefers `protocol`; covered by the wire tests. Severity: low.
- the `phases` read-property is not a Pydantic field, so it is absent from `model_dump()` (asserted) -- consumers that introspected `model_fields` for `phases` would not find it; none do. Severity: low.
- naming asymmetry: .NET wire is `CognitiveProtocolWire` but the downstream contract record is still `SportsCognitivePhases`. Documented as intentional; the downstream rename is deferred. Severity: low (cosmetic).
- the back-compat surfaces (class alias, `.phases` property, `phases` validation alias, .NET `Phases` alias) must be removed together in the later alias-removal slice; tracked. Severity: low.

### next recommended slice

The remaining migration-plan structural steps, now that naming is canonical at the analyzer surface: Stress collapse (plan Q1, isolated commit + calibration spot-check), then detect/aim array-vs-scalar (Q2), then v3 stamp (Q6) + legacy `CognitivePhases` persistence decision (Q3), then alias removal (step 7: drop the `phases` property/alias, the `SportsCognitivePhases` module alias, the wire `Phases` alias) and the downstream .NET contract record rename (`SportsCognitivePhases`->`SportsCognitiveProtocol` across `SportAnalysisContracts.cs`/builder/vocab-mapper). Confidence rules and posture enum stay untouched.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (FastAPI: 3 files; .NET: wire DTO + wire test) and `dai-vault` (this handoff). `jera-workspace-skills` untouched.
- Re-verify: FastAPI `\.venv\Scripts\python -m pytest tests/test_sports_analyzer.py` -> 97 passing; .NET `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 254 passing.
- The canonical Python member is `result.protocol`; `result.phases` is a read-only property alias kept for back-compat. New code should use `.protocol`.
- The .NET downstream contract is deliberately still `SportsCognitivePhases`/`SportsAnalysisResponse.Phases`; the wire layer (`CognitiveProtocolWire`, `Protocol` member) is what was renamed. Do NOT rename the downstream record until the dedicated slice (it touches CognitiveProtocolBuilder + ProtocolVocabularyMapper + persistence).
- Do NOT remove any compat alias/property, collapse Stress, or stamp v3 yet.
- Pre-existing untracked `dai-vault` calibration files are NOT from this slice; leave them.

status: protocol class/member rename merged 2026-05-26. Pydantic SportsCognitivePhases->SportsCognitiveProtocol (+alias); response member phases->protocol (+`.phases` property, +phases validation alias, serializes protocol); .NET wire CognitivePhasesWire->CognitiveProtocolWire with Protocol primary / Phases alias / protocol-preferred mapping; downstream .NET contract record intentionally unchanged. pytest 97 (+6), dotnet 254 (+1). v3/collapse/alias-removal/downstream-record-rename deferred. jera-workspace-skills untouched. COMMITTED+PUSHED: dai 0cc87b3 (feat: rename analyzer phases to protocol), dai-vault 181a408 (docs: document analyzer protocol rename).

## addendum: Decision Factory Hardening v1 -- Decision Artifact Contract v1 (2026-05-26)

Focus shift from operational production readiness to hardening how the decision artifact is manufactured. This first slice defines the internal artifact contract and begins enforcing it. No auth/billing/tenant/Kubernetes/deploy work. No confidence-math change, no DB schema change, no Angular change, no broad downstream contract rename, no persisted-shape change.

### naming and skills gate

Skills: `dai-grill-with-vault` (read SportsQualityChecker + its tests, SignalQualityEvaluator, SportsRetrievalOutput, the signal record shapes, and the sports-v1 product folder before deciding the slice -- this is what proved the anti-hype gate was safe against the fully-grounded clean-pass fixture), `superpowers:test-driven-development` (RED on rule 6 -> GREEN), `superpowers:systematic-debugging` (fixed a namespace import miss for SharpPublicLookupResult), `superpowers:verification-before-completion` (full dotnet + full pytest + ASCII check), `dai-token-tight`, `dai-agent-handoff`. `dai-signal-follow-up-diagnostics` considered (it is the closest runtime-doctrine skill) and used as background framing for the signal-availability/proxy reasoning, not invoked as a procedure. jera-workspace-skills untouched (no approval). Skill-fit note: `dai-signal-follow-up-diagnostics` is well-aligned with this hardening track; a companion `dai-decision-artifact-contract` skill (read the artifact contract, classify which of the 16 sections a complaint hits, propose the owning phase + gate) would fit future hardening slices.

Naming decisions (gate item documented): slice named **Decision Artifact Contract v1** per the brief. New quality gate is **rule 6 (anti-hype)**, keyed on the existing `ConfidenceEffect == "block_aggressive_posture"` vocabulary (no new signal vocabulary invented). Warning string is descriptive and stable: `aggressive posture=play asserted while signal layer blocks aggressive posture (missing confirmation signal)`. New test type named `DecisionArtifactContractTests` (characterization). Vault doc at the suggested path `04 Products/sports-v1/decision-factory-hardening-v1.md`. No type/field/contract renames this slice.

### files changed

- `dai-vault/04 Products/sports-v1/decision-factory-hardening-v1.md` (new) -- the durable hardening plan: north star, v1 success, phase responsibilities, the 16-section artifact contract mapped to owning phase + backing field + status, the 7-facet signal source strategy, proxy rules, quality gates (existing 5 + new rule 6 + planned), next slices. ASCII-clean.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsQualityChecker.cs` -- added rule 6 (anti-hype gate): warns when posture==play and any SignalAvailability record carries ConfidenceEffect==block_aggressive_posture. Reads the existing `retrieval.SignalAvailability`; additive warning only.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsQualityCheckerTests.cs` -- 3 rule-6 tests (warns on play+block; no-warn on play+confirmation-grounded; no-warn on cautious posture+block).
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/DecisionArtifactContractTests.cs` (new) -- 6 characterization tests pinning the contract: available/missing signals surfaced; weak signal graded usable-not-strong; strong only with confirmation; missing confirmation blocks aggressive posture.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No FastAPI change. No persisted-shape change. `jera-workspace-skills` untouched.

### behavior changes

One additive behavior change: a new deterministic `ArtifactQualityWarning` fires when an aggressive `play` posture is asserted while a confirmation signal is missing (the signal layer marks `block_aggressive_posture`). It surfaces the anti-hype risk on the inspection surface; it does NOT change the confidence number, the posture value, the customer DTO, or any persisted shape. Everything else is documentation + characterization tests that pin already-true guarantees so future hardening cannot silently regress them. Old persisted artifacts are unaffected (the gate runs at check time for new runs only).

### tests added / results

- dotnet test: 263 passed, 0 failed (was 254, +9: 3 rule-6 + 6 contract characterization). RED confirmed first on rule 6. Clean-pass and all existing rule tests stay green (rule 6 cannot fire on the fully-grounded fixture). Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:583` unrelated.
- pytest: 97 passed (unchanged; no FastAPI change this slice).
- serialization-compatible (testing req 5) and old-artifact-readable (req 6) are covered by the existing SportsAnalysisWireTests (97/254 wire round-trip) and ProtocolVocabularyMapperTests (v1 legacy fallback) suites; not duplicated here.

### risks

- rule 6 adds a warning in the play+missing-confirmation case; additive and inspection-only, but any external consumer that asserted an exact `ArtifactQualityWarnings` set for that scenario would see one more entry. No such consumer exists in-repo (verified by full-suite green). Severity: low.
- the gate keys on `block_aggressive_posture`, which today only `sharp_public`-missing produces; if future signals add that effect the gate widens automatically (intended). Severity: low.
- proxy explicit-labeling and runtime `confidence_high_for_partial_evidence` are documented as gaps, deliberately deferred (the latter intersects the confidence surface + clean-pass contract). Severity: low (tracked as next slices).

### next recommended slice

Proxy label surfacing (contract section 6 item): add an explicit proxy label to the artifact when a `lateral_proxy` informs the read, enforced by a quality gate, additive on the inspection surface. Then the injury/availability signal slice, then the runtime weak-evidence confidence gate.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (1 runtime file + 2 test files) and `dai-vault` (1 new product doc + this handoff). `jera-workspace-skills` untouched.
- Re-verify: .NET `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 263 passing; FastAPI `\.venv\Scripts\python -m pytest` -> 97 passing. No stack/DB/network needed (checker + evaluators are pure).
- The artifact contract is .NET-side and additive; the analyzer (FastAPI) already emits canonical `protocol`. Do NOT rename `SportsCognitivePhases`/`SportsAnalysisResponse.Phases` (deferred). Do NOT change confidence math or stamp v3 in hardening slices unless that is the explicit slice.
- rule 6 is the template for further deterministic gates: read the existing `SignalAvailability`/`SignalFollowUps` diagnostics, emit an additive `ArtifactQualityWarning`, fixture it so it cannot fire on the fully-grounded clean-pass case.
- Pre-existing untracked `dai-vault` calibration files and the out-of-pack `../home-*.png` / `lessons-learned.md` under the skills workdir are NOT from this slice; leave them.

status: Decision Artifact Contract v1 implemented 2026-05-26. vault contract doc + anti-hype quality gate (rule 6) + contract characterization tests. dotnet 263 (+9), pytest 97 (unchanged). no confidence/schema/Angular/contract-rename/persisted-shape change. next: proxy label surfacing. jera-workspace-skills untouched. COMMITTED+PUSHED: dai dcfc792 (feat(sports): add anti-hype quality gate and pin decision artifact contract), dai-vault cb9031e (docs(sports): define decision factory hardening v1 and artifact contract).

## addendum: Stress Collapse v1 (2026-05-28)

Code slice (TDD, Python + .NET). Plan Q1 structural change: the two legacy stress-adjacent fields (`interrogate.stress`, `discern.test`) collapse into a single canonical `discern.stress`. The analyzer prompt requests one field; the parser and the .NET wire DTO both accept old payloads via cross-block fallback (canonical -> legacy interrogate.stress -> legacy discern.test); the runtime contract is canonical-only; `watch_for` now derives from the collapsed source. No `phases`->`protocol` rename of the .NET downstream record this slice, no v3 stamp, no alias removal, no Probe/Synthesize change, no confidence-rule/schema/Angular/Tool Gateway/MCP/pgvector/Functions/Kubernetes change. Persisted v1/v2 records are never rewritten; their projection still produces a populated canonical `Stress` via the same wire collapse on the read side.

### naming and skills gate

Skills: `dai-grill-with-vault` (read the migration plan Q1, the analyzer prompt + `_parse_phases`/`_parse_response`, sports.py, SportAnalysisContracts.cs, SportsAnalysisWire.cs, CognitiveProtocolBuilder.cs, ProtocolVocabularyMapper.cs, CognitiveProtocolView.cs, and the relevant tests before deciding the seam), `superpowers:test-driven-development` (RED on the new collapse tests both languages -> GREEN after wire + parser + builder + mapper changes), `superpowers:verification-before-completion` (full pytest + full dotnet + ASCII check on the changed `.cs` + `.py` files, plus a prompt-region grep confirming the two legacy stress labels are gone and the canonical one is present), `dai-token-tight` (reporting), `dai-agent-handoff` (these notes). `dai-signal-follow-up-diagnostics` considered and skipped (signal coverage diagnosis, not contract collapse). jera-workspace-skills untouched (no approval). Skill-fit note carried forward: a `dai-implement-with-vault` skill suited to lockstep cross-language migration steps would still beat the interactive grill template here.

Naming decisions (gate item documented):
- canonical home for stress is `discern.stress` (matches the station registry id `discern.stress`, the vocabulary map, and `protocol-node-specs.md`). No new vocabulary.
- collapse seam lives at the wire/parser boundary, NOT in `CognitiveProtocolBuilder`: the wire (`SportsAnalysisResponseWire.ToModel()`) and the FastAPI `_parse_phases` resolve the cross-block fallback so the runtime contract `SportsCognitivePhases` is canonical-only (`SportsInterrogatePhase.Stress` removed; `SportsDiscernPhase.Test` removed; `SportsDiscernPhase.Stress` added). The builder simply copies `phases.Discern.Stress` through.
- fallback order is fixed and identical on both sides: 1. canonical `discern.stress`, 2. legacy `interrogate.stress`, 3. legacy `discern.test`. Single-source, never concatenated.
- the view shape `DiscernStressProtocolView` keeps its three fields (`LegacyInterrogateStress`, `LegacyDiscernTest`, `Canonical`) so the read-side DTO and the Angular `/dev/artifacts` page are untouched; the two legacy slots are now always null going forward (vestigial), and the value always rides in `Canonical`.
- compatibility aliases retained: Pydantic `SportsDiscernPhase.stress` carries `validation_alias=AliasChoices("stress", "test")` so a legacy `test` key in a discern block still binds; the .NET wire `DiscernPhaseWire.Test` stays for boundary acceptance; the wire `InterrogatePhaseWire.Stress` stays for cross-block legacy acceptance (documented in code as "legacy boundary only").

### files changed

- `dai/services/agent-service/app/services/sports_analyzer.py` -- `_JSON_SHAPE` drops `phases.interrogate.stress` and `phases.discern.test`, adds `phases.discern.stress` with its instruction text (merging fragility + condition-under-which-the-read-fails). Per-field instruction block updated to match. `counter_case`/`watch_for` cross-reference lines retargeted: `watch_for must match phases.discern.stress`. `_parse_phases` no longer reads `i.stress` or `d.test` into their old positions; instead it resolves the single canonical `stress = _s(d.get("stress")) or _s(i.get("stress")) or _s(d.get("test"))` and passes it to `SportsDiscernPhase(stress=stress)`. `_parse_response` `watch_for` derivation now reads `phases.discern.stress`.
- `dai/services/agent-service/app/models/sports.py` -- `SportsInterrogatePhase.stress` removed; `SportsDiscernPhase.test` removed; `SportsDiscernPhase.stress` added with `validation_alias=AliasChoices("stress","test")` and `populate_by_name=True`. Module-level comment block updated to describe the collapse and the cross-block resolution.
- `dai/services/agent-service/tests/test_sports_analyzer.py` -- `_make_full_phases` uses the canonical shape; `_CANONICAL_PHASES` drops the legacy stress keys and places `stress` under discern. Existing tests retargeted (`test_parse_response_phases_discern_stress_parses`, both `_PHASES` parse tests assert the collapsed `discern.stress`, model-validate tests assert the validation_alias). New section adds parser-side collapse tests with the explicit fallback order + a watch_for-derives-from-discern.stress test (+8 tests).
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs` -- `SportsInterrogatePhase` loses `Stress`; `SportsDiscernPhase` loses `Test` and gains `Stress`. Comments document the collapse and that the boundary fallback lives in the wire.
- `dai/platform/dotnet/DevCore.AiClient/SportsAnalysisWire.cs` -- `CognitiveProtocolWire.ToModel()` computes the cross-block `stress = discern.Stress ?? interrogate.Stress ?? discern.Test` and passes it into `discern.ToModel(stress)`. `InterrogatePhaseWire.Stress` retained as legacy-boundary input and is NOT mapped onto the interrogate model. `DiscernPhaseWire` gains a canonical `Stress` JSON property and a `ToModel(string? stress)` signature that places the resolved value onto `SportsDiscernPhase.Stress`. Legacy `Test` property retained as boundary input.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolBuilder.cs` -- the wire pre-resolves stress, so the builder copies `phases.Discern?.Stress` straight through. The previous `interrogate.Stress ?? discern.Test` fallback line is removed; doc comments updated to point at the wire seam.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/ProtocolVocabularyMapper.cs` -- v1 legacy projection now sources `Discern.Stress.Canonical` from `phases.Discern?.Stress` and sets both legacy slots to null. v2 canonical projection is unchanged (already canonical).
- `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolView.cs` -- doc comments updated to reflect single-source semantics; `DiscernStressProtocolView` shape unchanged so the read-side DTO and Angular dev page are unaffected; legacy slots flagged vestigial.
- .NET test updates: `SportsAnalysisWireTests` gains 5 collapse tests (canonical, legacy interrogate.stress, legacy discern.test, canonical-wins, interrogate-wins-over-test). `CognitiveProtocolBuilderTests` collapses its two "builder does the fallback" tests into one "builder copies canonical through" test plus a null guard. `ProtocolVocabularyMapperTests` retargets: legacy-projection puts the canonical stress in `Canonical` and leaves the vestigial slots null; the dual-legacy-source test becomes a vestigial-slots-null assertion. `SportsComposerTests.MakePhases` constructs the canonical record shape. `AgentRunsControllerTests` `legacy_phases_project_into_v1_response` retargeted: the contract record now carries `Discern.Stress` directly, so the assertions read from `Canonical` and confirm the two legacy slots are null.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No FastAPI prompt change beyond the three stress-related label edits. No persisted-shape change (the `OutputJson` `CognitiveProtocol` block still serializes the same canonical record; the `CognitivePhases` block's record fields are renamed in the wire/contract surface but the persisted-block legacy compatibility relies on the `v1 fallback` path that this slice has updated to surface canonical stress correctly). No Tool Gateway, MCP, pgvector, Azure, Angular, confidence-rule, or schema change. `jera-workspace-skills` untouched.

### behavior summary

The analyzer asks for one stress sentence under `discern.stress`. The parser collapses any incoming shape (canonical-only, legacy-only, or mixed) into a single canonical `SportsDiscernPhase.Stress` using the fixed fallback order (canonical -> legacy interrogate.stress -> legacy discern.test). The runtime contract `SportsCognitivePhases.Interrogate.Stress` is gone; the runtime contract `SportsCognitivePhases.Discern.Test` is gone. The `CognitiveProtocol` canonical record carries the single `Discern.Stress` value, and the read-side `DiscernStressProtocolView.Canonical` carries it on both v1 and v2 projection paths. `watch_for` derives from the same source. Old persisted records are never rewritten; their `CognitivePhases` block deserializes through the wire and the v1 projection path now surfaces the collapsed stress in `Canonical` as well, so reviewers reading the inspection endpoint see the single canonical value for any record going forward. The customer-facing `AgentRunResultDto` is unchanged.

### test results

- pytest (`tests/test_sports_analyzer.py`): 105 passed (was 97; +8). RED first on the collapse tests; GREEN after the parser + model changes.
- dotnet test (full suite): 266 passed, 0 failed (was 263; +3 wire collapse tests net after the builder/mapper test consolidation). RED first on the wire + builder collapse expectations; GREEN after the wire seam + builder simplification + view-comment update. Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:583` is unrelated.
- prompt grep confirms `phases.interrogate.stress` and `phases.discern.test` are gone from `_JSON_SHAPE` and `phases.discern.stress` is present.

### risks

- prompt content change (the model now writes one stress sentence under `discern` instead of two adjacent sentences across `interrogate` and `discern`) may shift calibration. The migration plan called for an isolated commit + calibration spot-check; the next live batch should be spot-checked against the prior baseline before any threshold change. Confidence math, posture enum, and the quality gates were not touched. Severity: med (calibration), low (correctness).
- `DiscernStressProtocolView` retains its two vestigial legacy slots; an external consumer that read `LegacyInterrogateStress` or `LegacyDiscernTest` directly would now see null on every record. No such consumer exists in-repo (verified by full-suite green plus the Angular dev page reading `Canonical` first). Severity: low.
- the .NET runtime contract `SportsCognitivePhases.Interrogate.Stress` and `Discern.Test` are gone; any external caller constructing these records by positional or named arguments must update. In-repo callers were updated this slice. Severity: low.
- old persisted v1 records whose `CognitivePhases.Interrogate.Stress` or `CognitivePhases.Discern.Test` were populated still deserialize -- but those legacy properties no longer exist on the deserialization target, so they are silently dropped at deserialization. The collapsed `Discern.Stress` is sourced from the same JSON payload via the wire fallback when the artifact is re-read, so the inspection projection still shows the value via `Canonical`. The persisted JSON itself is unchanged (we do not rewrite history). Severity: low.
- Pydantic `populate_by_name=True` is the older 2.x spelling (newer `validate_by_name`/`validate_by_alias`); a future pydantic major could deprecate it. Same risk as the alias scaffold slice, unchanged. Severity: low.

### next recommended slice

Plan Q2: `perceive.detect` and `perceive.aim` array-vs-scalar decision (recommended in the plan to keep arrays to avoid prompt churn; either way it is its own isolated commit). Then Q6 the `sports_decision_artifact_v3` stamp + Q3 the legacy `CognitivePhases` persistence decision; then plan step 7 alias removal (drop Pydantic `phases` property/alias, the `SportsCognitivePhases` module alias, the wire `Phases`/`Test`/cross-block `InterrogatePhaseWire.Stress` aliases) together with the downstream .NET contract record rename (`SportsCognitivePhases` -> `SportsCognitiveProtocol` across `SportAnalysisContracts.cs`/builder/vocab-mapper). Confidence rules and posture enum stay untouched throughout.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (FastAPI: 3 files; .NET: 4 runtime files + 5 test files) and `dai-vault` (this handoff). `jera-workspace-skills` read-only, untouched.
- Re-verify: FastAPI `\.venv\Scripts\python -m pytest tests/test_sports_analyzer.py` -> 105 passing; .NET `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 266 passing. No stack/DB/network needed (wire round-trips and parser/builder logic are pure).
- Collapse seam: the wire (`CognitiveProtocolWire.ToModel()`) and the FastAPI `_parse_phases` are the ONLY places that read legacy `interrogate.stress` or `discern.test`. Do not reintroduce a builder-side fallback; the builder is canonical pass-through now.
- The vestigial `DiscernStressProtocolView.LegacyInterrogateStress` and `LegacyDiscernTest` slots are retained for the read-side DTO/Angular contract; they are always null and should be removed together with the downstream .NET contract record rename in plan step 7. Until then do not change the view shape.
- The untracked `dai/scripts/dev/sports/purge-dev-agent-runs.ps1` is NOT part of this slice (separate dev tool; leave it). Pre-existing untracked `dai-vault` calibration files under `04 Products/sports-v1/calibration/` are NOT from this slice either; leave them.
- No PowerShell changed this slice, so no ASCII/parser-validation step was required beyond the ASCII check on the changed `.cs` and `.py` files.

status: Stress Collapse v1 implemented 2026-05-28. analyzer prompt emits one canonical `discern.stress`; FastAPI parser + .NET wire collapse legacy `interrogate.stress` / `discern.test` at the boundary (canonical -> interrogate.stress -> discern.test); runtime contract canonical-only; builder/mapper/view simplified; `watch_for` sources from canonical. pytest 105 (+8), dotnet 266 (+3). next: detect/aim array-vs-scalar (Q2), then v3 stamp + alias removal. jera-workspace-skills untouched. COMMITTED+PUSHED: dai 51163a2 (feat(protocol): collapse stress to single discern source), dai-vault 789df07 (docs(protocol): document stress collapse v1).

## addendum: Perceive Detect/Aim Scalar Collapse v1 (2026-05-28)

Code slice (TDD, Python + .NET). Plan Q2: `perceive.detect` and `perceive.aim` collapse from legacy arrays into canonical scalar station outputs. Approved direction: station outputs are scalar; multi-fact retrieval belongs in structured signal fields, not in perceive. The analyzer prompt requests one scalar string per field; both boundaries (the .NET wire/persisted-read converter and the Pydantic BeforeValidator) accept either a scalar string or a legacy `list[str]` (joined with `"; "`) so old persisted v1/v2 OutputJson records and any in-flight legacy payload still parse. The runtime contract is canonical scalar; `CognitiveProtocolBuilder` is now pure pass-through for perceive (the `JoinArray` helper is removed). The view shape (`PerceiveProtocolView.Detect`/`.Aim` as `string[]?`) is intentionally retained so the Angular `/dev/artifacts` page is untouched; `ProtocolVocabularyMapper.ProjectFromLegacy` splits the scalar back on the same `"; "` boundary, round-tripping multi-element legacy records into the existing array view. No v3 stamp, no alias removal, no confidence/posture/schema/Tool Gateway/MCP/pgvector/Azure/Kubernetes change. Persisted records are never rewritten.

### naming and skills gate

Skills used (jera pack, local): `dai-grill-with-vault` (read the migration plan Q2, the vocabulary map, the station blueprint, the canonical `CognitiveProtocol`, the legacy contract record + wire DTO + builder + mapper + view + Pydantic models + prompt + parser, and every existing perceive test BEFORE naming or coding -- the read found that the canonical `CognitiveProtocol.PerceiveProtocol` is already scalar and that the only consumers of legacy perceive arrays are the builder's `JoinArray` and the mapper's `SplitJoined`, which inverted into the simple boundary-converter design); `dai-token-tight` (reporting density); `dai-agent-handoff` (these transfer notes).

Skills used (superpowers / Claude built-ins): `superpowers:test-driven-development` (RED on the new collapse expectations both languages -> GREEN after wire + parser + builder + mapper changes), `superpowers:verification-before-completion` (full pytest + full dotnet + ASCII check on the changed `.cs` and `.py` files, plus a prompt-region grep confirming the scalar JSON shape replaced the array shape and the `do not write a list` guardrail is present), `superpowers:writing-plans` (consulted -- the migration plan already specifies Q2; this gate confirmed shapes and locked the boundary tolerance design instead of replanning), `superpowers:systematic-debugging` (held in reserve; not needed -- no regression to chase). `superpowers:brainstorming` was not used because the direction was approved; `superpowers:planning` was not used because the migration plan already exists. `dai-grill-me`, `dai-write-skill`, `dai-signal-follow-up-diagnostics` considered and skipped (no fuzzy plan to interrogate; no skill to write; no signal-coverage gap to diagnose).

Skill-fit note carried forward: `dai-grill-with-vault` again did the read-and-verify job but its interactive closing template still does not match a solo implementation slice. The recommendation for a dedicated `dai-implement-with-vault` (or `dai-audit`) skill that documents which of the 16 cognitive station outputs a complaint hits, proposes the smallest shape-collapse, and produces a handoff-shaped closing summary stands. `jera-workspace-skills` left untouched this slice (no approval to edit -- repo boundary rule honored).

Naming and shape review result (gate item documented):
- canonical home stays at `perceive.detect` and `perceive.aim` -- already the station ids, vocabulary-map entries, and `CognitiveProtocol.PerceiveProtocol.Detect`/`.Aim` slots (see `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocol.cs:34-38`). No new vocabulary.
- the migration plan's Q2 recommendation (keep arrays, revisit later -- `fastapi-canonical-field-migration-plan-v1.md:184`) is **overridden by approved user direction**. Reason: detect/aim are decision-stage station outputs, not raw signal lists; structured signal records own multi-fact retrieval. The override is captured in this addendum; the plan doc is left as-is so the prose chronology of the decision is intact.
- legacy boundary tolerance via **one .NET converter + one Pydantic BeforeValidator**, not per-call shims. The .NET converter `StringOrStringArrayJsonConverter` is applied via `[JsonConverter]` attribute on the two scalar properties of both `SportsPerceivePhase` (contract record, covers persisted-reread) and `PerceivePhaseWire` (wire DTO, covers inbound FastAPI). The Pydantic `PerceiveScalar = Annotated[str | None, BeforeValidator(_coerce_str_or_str_list)]` covers `SportsPerceivePhase.detect`/`.aim`. Both apply the same coercion rules: `str` (trimmed; whitespace -> None), `list[str]` (non-empty trimmed entries joined with `"; "`; `[]` -> None), `None` -> None, anything else -> None.
- builder becomes pure pass-through for perceive; `JoinArray` helper is removed from `CognitiveProtocolBuilder.cs`. The reason "join with '; '" lives now at the wire/persisted-read boundary, not in the builder.
- view shape is **retained as `string[]?`** so the Angular `/dev/artifacts` contract is unchanged. `ProtocolVocabularyMapper.ProjectFromLegacy` switches from passing the legacy `string[]` directly to calling `SplitJoined` on the (now-scalar) legacy `string?`. An old persisted v1 record whose JSON has `"Detect": ["a", "b"]` collapses to `"a; b"` in the runtime contract on deserialize, then projects back as `["a", "b"]` in the view via `SplitJoined`. No Angular code touched, no view JSON shape change.
- prompt instructions flip to scalar in `_JSON_SHAPE` and the per-field instruction block: `"detect": "one sentence: ..."`, `"aim": "one sentence: ..."`, plus explicit guardrails ("do not write a list", "single attention focus is the canonical station output", "single aim is the canonical station output", "multi-fact retrieval lives in the signal layer, not in perceive").
- no v3 stamp this slice. The legacy `CognitivePhases.Perceive` block on new persisted records now serializes scalar strings, but the canonical `CognitiveProtocol` block is unchanged (it was already scalar). Reading any older `CognitivePhases` record with array shape still works via the contract-record converter. v3 stamping is migration plan Q6, deferred.

Vault / doc updates needed: this addendum. Migration plan Q2 entry stays prose-historical; the override is documented here rather than mutating the plan doc.

### files changed

- `dai/platform/dotnet/DevCore.AiClient/StringOrStringArrayJsonConverter.cs` (new) -- `JsonConverter<string?>` accepting `null`, scalar `string`, or `string[]`. Arrays join non-empty trimmed entries with `"; "`. Designed as the .NET parity of `_coerce_str_or_str_list`.
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs` -- `SportsPerceivePhase.Detect` and `.Aim` go from `string[]` to `string?` with `[JsonConverter(typeof(StringOrStringArrayJsonConverter))]` per property. Comment block documents the collapse and the boundary semantics.
- `dai/platform/dotnet/DevCore.AiClient/SportsAnalysisWire.cs` -- `PerceivePhaseWire.Detect` and `.Aim` go from `string[]?` to `string?` with the same converter attached. `ToModel()` becomes `new(Detect, Frame, Aim)`. Comment block documents the wire-boundary collapse.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolBuilder.cs` -- `JoinArray` helper removed; the perceive block becomes pure pass-through (`Detect: phases.Perceive?.Detect`, `Aim: phases.Perceive?.Aim`). Mapping-rule comment updated to point at the wire/persisted-read seam.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/ProtocolVocabularyMapper.cs` -- `ProjectFromLegacy` now calls `SplitJoined` on perceive Detect/Aim so the view's array shape is preserved for old persisted records. `JoinSeparator` doc comment updated to point at the wire/persisted-read boundary as the join site.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolView.cs` -- doc comment on `PerceiveProtocolView` updated to explain that the view keeps `string[]?` to leave the Angular contract untouched while the runtime is scalar.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/CognitiveProtocolBuilderTests.cs` -- the "flattens" perceive test becomes a "copies through" test; the empty-arrays test becomes a null-scalars test; the `BuildPhases` default uses `new SportsPerceivePhase(null, null, null)`.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/ProtocolVocabularyMapperTests.cs` -- legacy constructors flip to scalars; new test for legacy multi-element scalar round-tripping back into the view's array shape via `SplitJoined`; new null-scalar test; `BuildPhases` default updated.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsAnalysisWireTests.cs` -- the legacy-payload test now asserts the joined scalar (`"a; b"`); 6 new tests cover canonical scalar pass-through, legacy array join for detect and for aim, empty arrays -> null, null fields stay null, canonical winning when both `protocol` and `phases` blocks are present, and a persisted-record reread test that round-trips through the contract converter directly (no wire DTO).
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsComposerTests.cs` -- `MakePhases` uses scalar perceive (`"home momentum"`, `"momentum"`). Assertions about `protocol.Perceive.Detect`/`Aim` unchanged.
- `dai/platform/dotnet/DevCore.Api.Tests/Integration/AgentRunsControllerTests.cs` -- `MakePhases` uses scalar perceive. View assertions unchanged (the view still serves `string[]?` via `SplitJoined`).
- `dai/services/agent-service/app/models/sports.py` -- adds `PerceiveScalar = Annotated[str | None, BeforeValidator(_coerce_str_or_str_list)]` plus the helper. `SportsPerceivePhase.detect` and `.aim` flip from `list[str]` to `PerceiveScalar = None`.
- `dai/services/agent-service/app/services/sports_analyzer.py` -- prompt `_JSON_SHAPE` shows scalar one-sentence slots; per-field instructions explain station-output semantics and prohibit list / multi-sentence phrasing. Parser drops `_sl` in favor of a new `_s_or_list` helper that mirrors the .NET converter rules; the perceive constructor call uses it.
- `dai/services/agent-service/tests/test_sports_analyzer.py` -- `_make_full_phases` and `_CANONICAL_PHASES` use scalar perceive; `_LEGACY_PHASES` keeps arrays. `test_..._perceive_detect_parses_as_list` / `..._aim_parses_as_list` retargeted to `..._as_scalar`. The two existing canonical/legacy parse tests gain perceive-shape assertions. 12 new tests cover canonical scalar pass-through, legacy array join (detect and aim separately), empty arrays -> None, null stays None, blank-entry filtering, Pydantic model accepting scalar directly, Pydantic model accepting legacy list via the BeforeValidator, empty-list -> None at the model layer, a model-fields annotation guard against list regressions, the new scalar prompt instructions, and the retirement of the old "list 1 to 3" prompt phrasing.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No FastAPI prompt change beyond the perceive-related label / instruction edits. No persisted `OutputJson` schema change. No Tool Gateway, MCP, pgvector, Azure, Angular, confidence-rule, posture-enum, or DB schema change. `jera-workspace-skills` untouched.

### behavior summary

The analyzer asks for one scalar station-output sentence for `perceive.detect` and one for `perceive.aim`. The .NET wire DTO and the FastAPI parser accept either a scalar string or a legacy `list[str]` (joined with `"; "` into a single canonical scalar). The runtime contract `SportsCognitivePhases.SportsPerceivePhase.Detect`/`.Aim` is scalar (`string?`). `CognitiveProtocolBuilder` copies the scalar straight through into `CognitiveProtocol.PerceiveProtocol.Detect`/`.Aim`. The persisted artifact's canonical `CognitiveProtocol` block is unchanged in shape; the persisted legacy `CognitivePhases` block now serializes scalar strings on new runs (and continues to deserialize older array shapes via the contract-record converter). The read-side projection (`ProtocolView`) and the artifact endpoint still surface `string[]?` for `Perceive.Detect`/`.Aim` -- the Angular `/dev/artifacts` page sees the same view shape as before. `watch_for`, `counter_case`, the posture enum, confidence math, and the deliver-layer extracts are unchanged.

### compatibility behavior

- canonical scalar payload (new prompt): passes straight through both boundaries to the canonical scalar runtime record.
- legacy multi-element array payload (old prompt, an old persisted record, or any in-flight legacy traffic during the window): joined with `"; "` at the boundary into a single canonical scalar. The view's array shape is restored by `SplitJoined` on read, so the dev page renders identically for old records.
- legacy empty array: becomes `null`/`None` on the scalar field (not the empty string).
- mixed canonical + legacy keys on the same block: `protocol` (canonical block) wins over `phases` (legacy block) at the response level; within whichever block is read, the converter/BeforeValidator coerce whatever shape is present.
- old persisted v1 records continue to deserialize directly into the new scalar contract because the `[JsonConverter]` is on the record property itself, not just on the wire DTO.

### test results

- pytest (`tests/test_sports_analyzer.py`): 117 passed (was 105; +12). RED first on the new canonical / legacy-array / shape-guard tests; GREEN after the model + parser + prompt changes.
- dotnet test (full suite): 275 passed, 0 failed (was 266; +9 net, after the builder test consolidation). RED first on the wire boundary + builder pass-through + projection round-trip expectations; GREEN after the converter + contract record + wire DTO + builder + mapper changes. Pre-existing `xUnit2013` warning at `AgentRunsControllerTests.cs:584` is unrelated.
- prompt grep confirms `"detect": "one sentence` and `"aim": "one sentence` are present in `_JSON_SHAPE` and the old `"list 1 to 3 short attention triggers"` / `"list 1 to 3 primary factors"` instructions are gone.

### risks

- prompt content change (the model now writes one scalar sentence per perceive station instead of a list of 1-3) may shift the texture of perceive outputs. The migration plan called for keeping arrays partly to avoid this risk; the user override is explicit. Confidence math, posture enum, and quality gates are untouched, so calibration thresholds are unaffected; the next live batch should be spot-checked against the prior baseline before any threshold change. Severity: med (texture), low (correctness).
- the `StringOrStringArrayJsonConverter` accepts any scalar `string`. If the model ever emits a multi-paragraph block for `detect`/`aim`, the converter takes it verbatim. The prompt guardrail ("do not write a list, multiple sentences, or bullet-style phrasing; a single attention focus is the canonical station output") is the only push-back. A future deterministic clamp (truncate to one sentence) could be added, but is out of scope this slice. Severity: low.
- old persisted v1 records whose `CognitivePhases.Perceive.Detect` or `.Aim` were non-empty `string[]` continue to deserialize via the contract-record converter; the canonical scalar value is the `"; "`-joined version of that array. The dev page renders the same multi-element array via `SplitJoined`. The persisted JSON on disk is unchanged (we do not rewrite history). Severity: low.
- the `PerceiveProtocolView` shape stays `string[]?`. An external consumer that begins relying on a scalar field would see no change. Angular is not touched. Severity: low.
- the `PerceiveScalar` annotation uses Pydantic v2 `Annotated[..., BeforeValidator]` semantics; this is the supported public API in 2.12.5 (installed) and is forward-compatible. Severity: low.

### next recommended slice

Migration plan steps 6 + 7 + the downstream rename, in roughly this order: (a) Q6 -- stamp `sports_decision_artifact_v3` on canonical-native runs, with (b) Q3 -- the legacy `CognitivePhases` persistence decision (recommend dropping the legacy block on v3 records; old records remain readable via `ProtocolVocabularyMapper.ProjectFromLegacy`); then (c) step 7 -- alias removal (drop the Pydantic `phases` property/alias, the `SportsCognitivePhases` module alias, the wire `Phases` / `Test` / cross-block `InterrogatePhaseWire.Stress` aliases, and the perceive `[JsonConverter(StringOrStringArrayJsonConverter)]` once a calibration window has confirmed no legacy array payloads remain); then (d) the downstream .NET contract record rename `SportsCognitivePhases` -> `SportsCognitiveProtocol` across `SportAnalysisContracts.cs` / builder / vocab-mapper. Confidence rules and posture enum stay untouched throughout. Calibration spot-check on the next live batch is prudent before (a).

### final git status snapshot

- `dai`: 13 modified files + 1 new file (`StringOrStringArrayJsonConverter.cs`) staged for this slice; the unrelated untracked `scripts/dev/sports/purge-dev-agent-runs.ps1` is **not** part of this slice (left untracked, same as in Stress Collapse v1).
- `dai-vault`: this addendum is the only modification owned by this slice; the pre-existing modified `04 Products/sports-v1/calibration/20260508-1059-nba-calibration.md` and the untracked NBA calibration md + artifacts json files under `04 Products/sports-v1/calibration/` are **not** from this slice (left as-is, same as in Stress Collapse v1).
- `jera-workspace-skills`: read-only this slice; no edits. Confirmed clean against the local pack at `C:/Users/trolo/source/repos/jera-workspace/jera-workspace-skills`.

### Claude <-> Codex transfer notes

- Repos in play: `dai` (DevCore.AiClient: 1 new converter + 2 modified files; DevCore.Api: 3 runtime files modified; DevCore.Api.Tests: 5 test files modified; agent-service: 3 files modified) and `dai-vault` (this handoff). `jera-workspace-skills` untouched.
- Re-verify anywhere: FastAPI `\.venv\Scripts\python -m pytest tests/test_sports_analyzer.py` -> 117 passing; .NET `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 275 passing. No stack/DB/network needed (wire round-trips, parser/builder logic, and converter behavior are pure).
- The boundary collapse seam: `StringOrStringArrayJsonConverter.cs` for .NET and `_coerce_str_or_str_list` (Pydantic `BeforeValidator`) for FastAPI. Do NOT reintroduce a builder-side join; the builder is pure pass-through now.
- The `DiscernStressProtocolView` legacy slots from Stress Collapse v1 remain vestigial. The `PerceiveProtocolView.Detect`/`.Aim` stay `string[]?` for Angular -- do NOT change without an explicit Angular slice.
- The untracked `dai/scripts/dev/sports/purge-dev-agent-runs.ps1` is NOT part of this slice. Pre-existing untracked `dai-vault` calibration files under `04 Products/sports-v1/calibration/` are NOT from this slice either. Leave both.
- No PowerShell changed this slice, so no ASCII/parser-validation step was required beyond the ASCII check on the changed `.cs` and `.py` files (clean -- comments are lowercase/ascii per CLAUDE.md).
- Skill weakness / sharpening recommendations to carry forward: (1) `dai-grill-with-vault`'s closing template still does not match a solo implementation slice -- a dedicated `dai-implement-with-vault` (or `dai-audit`) skill would fit better; (2) a future skill could include a "do not blanket replace_all dotted strings that appear as both prompt labels and attribute access" guard (the Canonical Prompt Pure Rename v1 footgun); (3) a future skill could codify the migration-step pattern "shape change -> add boundary converter -> update contract type -> remove builder helper -> add view round-trip if Angular contract is unchanged".

status: Perceive Detect/Aim Scalar Collapse v1 implemented 2026-05-28. analyzer prompt emits scalar `perceive.detect` / `perceive.aim`; both boundaries collapse legacy `string[]` payloads via `"; "`-join (StringOrStringArrayJsonConverter on the .NET side, BeforeValidator on the Pydantic side); runtime contract canonical-only scalar; builder pass-through; view shape retained as `string[]?` so Angular is untouched; persisted records never rewritten. pytest 117 (+12), dotnet 275 (+9). next: artifact v3 stamp (Q6) + legacy block persistence decision (Q3) + alias removal (step 7) + downstream contract rename. jera-workspace-skills untouched. COMMITTED+PUSHED: dai c7342e6 (feat(protocol): collapse perceive detect aim to scalar fields), dai-vault 3126827 (docs(protocol): document perceive scalar collapse).

## addendum: Canonical Protocol Calibration Spot-Check v1 (2026-05-28)

Observational slice (live runs, no code change). Confirmed the canonical Cognitive Protocol Runtime is healthy end-to-end after the recent migration arc (Canonical Prompt Pure Rename v1 -> `phases -> protocol` rename -> Stress Collapse v1 -> Perceive Detect/Aim Scalar Collapse v1). Three live NBA runs through the full pipeline (reference, retrieve, analyze, compose, persist) produced canonical-shaped artifacts; inspection endpoint and the `protocolView` projection serve cleanly; no first-class runtime dependency remains on the retired field names. No confidence/posture/Tool-Gateway/schema/Angular/MCP/pgvector/Azure/Kubernetes/secrets change; **no v3 stamp landed this slice** (it is the recommended next slice).

### naming and skills gate

Skills used (jera pack, local, read-only):
- `dai-grill-with-vault` -- read the migration plan Q2 + Q6 entries, the cognitive-factory docs (vocabulary map, station blueprint, node specs), the harness source (`run-artifact-calibration.ps1`), prior calibration notes (2026-05-18 NBA and MLB), and the artifact endpoint contract before bringing up the stack; surfaced the harness markdown reporter's reading of retired `cognitivePhases.interrogate.stress` / `cognitivePhases.discern.test` as a known reporter divergence, not a runtime bug.
- `dai-token-tight` -- reporting density on the spot-check note and these transfer notes.
- `dai-agent-handoff` -- shape of these transfer notes.

Skills used (superpowers / Claude built-ins):
- `superpowers:verification-before-completion` -- every canonical-shape claim grounded in a direct read of the three produced artifact JSONs and the live artifact endpoint response; the `protocolView.discern.stress.Canonical` slot was inspected on a real run; the legacy slots were confirmed null end-to-end. No claim was extrapolated from the test suite alone.
- `superpowers:writing-plans` -- consulted to keep the spot-check sequence minimal and revertible: stand up stack, run a small batch, inspect, write the note, no code touched.
- `superpowers:systematic-debugging` -- held in reserve; not needed.
- `superpowers:planning` -- consulted only enough to confirm v3 is the right next slice; the migration plan already specifies it (Q6).

Skill weakness / sharpening recommendations carried forward:
1. The harness markdown reporter is the next sharpening surface. A small `dai-calibration-report-canonical` slice (and matching skill) should: read the canonical `cognitiveProtocol` block first, fall back to `cognitivePhases` only for older records, drop the retired field columns (`interrogate.stress`, `discern.test`) from the table, and add the new canonical columns (`interrogate.probe`, `discern.stress.canonical`). One file, no runtime change.
2. A dedicated `dai-implement-with-vault` (or `dai-spot-check-with-vault`) skill that codifies this exact spot-check shape -- live run + JSON facet table + calibration delta vs prior batch + v3-readiness conclusion + no-code outcome -- would fit operational verification slices better than the interactive grill template.
3. `jera-workspace-skills` left untouched (repo boundary rule honored, no approval to edit). Confirmed clean at `C:/Users/trolo/source/repos/jera-workspace/jera-workspace-skills`.

Naming decisions (gate item documented):
- spot-check note filename: `20260528-0931-nba-canonical-protocol-spot-check.md` -- timestamp prefix matches the harness's auto-generated report so they sort together; suffix is descriptive without overloading the harness's own naming convention.
- the note sits in `dai-vault/04 Products/sports-v1/calibration/` next to the harness output (matches the existing calibration area; no new folder).
- harness-produced sibling files (auto-named by the script, kept verbatim): `20260528-0931-nba-calibration.md` + three artifact JSONs under `artifacts/20260528-0931-nba-*.json` (canonical names from `run-artifact-calibration.ps1`; no rename).
- v3 stamp constant name (proposed, not yet committed): `ArtifactVersions.SportsDecisionArtifactV3 = "sports_decision_artifact_v3"` -- mirrors the existing V2 constant; vocabulary already approved in the migration plan Q6.

### artifact shape findings (all 3 runs)

Canonical `cognitiveProtocol` block:
- `cognitiveProtocol` present on every run; `artifactVersion = sports_decision_artifact_v2` (v3 deliberately not stamped).
- `perceive.detect` and `perceive.aim` are scalar `string` on every run -- the model writes one sentence per station, no list semantics, no semicolon-joined multi-fact phrasing; the Perceive Scalar Collapse v1 prompt guardrail is being honored at emission.
- `interrogate` keys are exactly `question`, `probe`, `verify` -- `stress` is absent (Stress Collapse v1 honored at the model boundary, not just at the parser).
- `interrogate.probe` is non-null on every run because `sharp_public` is missing and the doctrinal probe template fires; the deterministic `BuildProbe(SignalFollowUpRecord[])` path is producing real material.
- `discern` keys are exactly `weigh`, `contrast`, `stress` -- `test` is absent (single canonical stress source per Collapse v1).
- `discern.stress` is a single sentence on every run.
- `decide.position` carries the validated posture enum value (`monitor` on every run) verbatim.
- `synthesize.{integrate, compose, deliver}` all present.

Legacy compatibility block (`cognitivePhases`, still emitted; Q3 not yet decided): scalar perceive detect/aim, interrogate carries `balance`/`reframe` (no `stress`), discern carries `filter`/`listen`/`stress` (no `test`). The dual-emit invariant still holds.

`protocolView` projection (the read-side surface served to `/dev/artifacts`):
- `perceive.detect`/`.aim` come through as `string[]` with 1 element -- the canonical scalar wrapped by `SplitJoined` (Angular contract preserved by design per Perceive Scalar Collapse v1).
- `interrogate.probe` is populated.
- `discern.stress` is a `DiscernStressProtocolView` with `canonical` populated and both legacy slots null -- end-to-end proof of Stress Collapse v1.
- `synthesize` carries all three platform-operational descriptions.

### quality warning findings

Runtime `artifactQualityWarnings` was empty on all three runs:
- the anti-hype gate (rule 6, Decision Artifact Contract v1) did not fire because every run was `posture == "monitor"`; the gate requires `play` + `block_aggressive_posture` and correctly stayed silent.
- signal-narrative drift and `signals_used` integrity checks were clean.
- posture validation: every run inside the validated enum, no clamp.

Offline calibration flags (computed by the harness markdown report; distinct from runtime `ArtifactQualityWarnings` per the 2026-05-21 audit): `missing_sharp_public` 3/3, `signal_quality_blocks_aggressive_posture` 3/3, `posture_aligned_with_partial_evidence` 3/3, `counter_case_generic` 2/3, `what_would_change_contains_filler` 1/3, `frame_missing_rest_context` 1/3, `confidence_high_for_partial_evidence` 0/3 (no run reached confidence >= 0.70 -- the prior batch's 0.72 baseline did not repeat here).

### Tool Gateway / telemetry observations

The runs completed end-to-end through the canonical chain (`schedule.matchup_dates` at `platform.reference`; `schedule.basketball.rest_context`, `market.basketball.spread`, `market.sharp_public.split` at `platform.retrieve`; `analysis.sports.matchup_read` at `platform.analyze`). Success at every stage proves the `ProtocolRegistry` startup guard cross-check (Protocol Registry Startup Guard v1) was satisfied at boot and that the scoped DI graph is wiring keyed handlers correctly under load. Structured `ToolGatewayInvocation` log events are emitted per Tool Gateway Correlation and Telemetry v1; this spot-check did not capture live stdout from the .NET API window, but the telemetry path is covered by 4 dedicated tests in the 275-test .NET suite. No `ToolNotAllowed`, `ToolNotRegistered`, or `HandlerError` outcomes inferred from the run results.

### calibration delta vs prior NBA batch

Sample is small on both sides (2 vs 3); treat directionally. Posture distribution identical (all `monitor`). Confidence range lower this batch (0.63 -- 0.675 vs 0.72 prior). Evidence richness identical (2 every run). `confidence_high_for_partial_evidence` dropped from 2/2 to 0/3 because confidence is below 0.70, not because the rule changed. `interrogate_unsupported_claim` dropped to 0/3 from 1/2 -- the canonical prompt may be tightening interrogate quality. `counter_case_generic` rose to 2/3 from 0/2 -- could be a prompt-tightening signal or a sample-size artefact. Next calibration window should aim for take 8+ across NBA and MLB before any threshold change.

### reporter divergence (known, not a runtime issue)

The harness markdown report's `Cognitive Phase Completeness` and `Cognitive Phase Quality` tables show `not recorded` for `interrogate.stress` and `discern.test` on every new run, because the harness reads `cognitivePhases.interrogate.stress` and `cognitivePhases.discern.test`, both of which are intentionally retired after Stress Collapse v1. The actual stress content is in `cognitivePhases.discern.stress` (shown as `recorded`), and the canonical home is `cognitiveProtocol.discern.stress` (the report does not inspect it at all). Reporter limitation, not a runtime regression. Out of scope for this spot-check; carry as a small follow-up slice.

### risks

- the prompt-texture shift (lower confidence range, fewer unsupported-claim flags, more generic counter_case) might mask a real quality regression on `counter_case`. Mitigation: capture another 5-8 run batch within a week and reassess. Severity: low-med.
- the harness reporter's "not recorded" entries for retired field names could mislead a casual reader. Mitigation: scheduled as the follow-up slice. Severity: low.
- v2 records (this batch) and v3 records (after the next slice) will coexist in inspection. The mapper preserves both projection paths; `protocolView` will continue to render correctly for both. No data migration. Severity: low.

### is v3 stamp now safe?

Yes. The canonical analyzer is emitting the v3-shaped artifact today (the only thing left to do is mark it with the constant). Every facet of the canonical contract is healthy on every run; the deterministic Probe and Synthesize layers are producing real material; `protocolView.discern.stress.Canonical` is populated and the legacy slots are null end-to-end. The v3 stamp is a marker, not a behavior change; no confidence or posture threshold moves.

### files changed

- `dai-vault/04 Products/sports-v1/calibration/20260528-0931-nba-canonical-protocol-spot-check.md` (new, this slice's primary artifact) -- the spot-check note: shape facets, calibration delta, reporter divergence, v3 readiness conclusion, skills used.
- `dai-vault/04 Products/sports-v1/calibration/20260528-0931-nba-calibration.md` (new, auto-generated by the harness) -- the harness's own markdown report for the batch.
- `dai-vault/04 Products/sports-v1/calibration/artifacts/20260528-0931-nba-f48d433e.json`, `...-fa8d433e.json`, `...-018e433e.json` (new, auto-generated by the harness) -- per-run artifact JSONs.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No `dai` repo changes. No `jera-workspace-skills` changes.

### next recommended slice

Artifact v3 Stamp v1 (migration plan Q6) + legacy `CognitivePhases` persistence decision (migration plan Q3), in one small commit. Recommended sequence:

1. add `ArtifactVersions.SportsDecisionArtifactV3 = "sports_decision_artifact_v3"`.
2. stamp v3 on the success-path `SportsComposer` result.
3. decide and apply the legacy `cognitivePhases` persistence policy on v3 records (recommend dropping the legacy block on v3 because the read-side projection already prefers canonical and old records remain readable via `ProjectFromLegacy`).
4. update the artifact-version label on `/dev/artifacts` and the source-badge text to recognize v3 (Angular text only; no contract change).
5. small dotnet suite update (artifact-version + composer tests); pytest unaffected.

After that: plan step 7 (alias removal, including the perceive `StringOrStringArrayJsonConverter` once a calibration window confirms no legacy array payloads remain) and the downstream `.NET` contract record rename (`SportsCognitivePhases -> SportsCognitiveProtocol`).

Parallel candidate (separate, smaller): teach `run-artifact-calibration.ps1` to read the canonical `cognitiveProtocol` block first and fall back to `cognitivePhases` for older records, closing the reporter divergence.

### Claude <-> Codex transfer notes

- Repos in play: `dai-vault` (1 new spot-check note + 1 harness markdown + 3 artifact JSONs + this addendum). `dai` and `jera-workspace-skills` untouched.
- Stack state at the end of this slice: Docker Desktop running, `devcore-sql` container up, `.NET` API on `:5007` and FastAPI on `:8000` running in separate PowerShell windows. The next agent can either continue with the v3 stamp slice (no stack needed -- pure code + tests) or stop the stack via `scripts/dev/sports/stop-sports-dev.ps1`.
- To re-verify the canonical shape on any machine: run the harness with `-Force` against the current stack, then inspect one of the produced artifact JSONs for `cognitiveProtocol.perceive.detect` (must be `string`), `cognitiveProtocol.interrogate` keys (must be `["question","probe","verify"]`), `cognitiveProtocol.discern` keys (must be `["weigh","contrast","stress"]`), and `protocolView.discern.stress.canonical` (must be non-null with the legacy slots null on the same call).
- The harness markdown reporter is known-stale on `interrogate.stress` and `discern.test`. Do NOT interpret its "not recorded" entries on those columns as a runtime regression. The canonical truth is in the JSON.
- Pre-existing untracked `dai-vault` calibration files (2026-05-08 / 05-09 / 05-10 NBA + their artifact JSONs) are NOT from this slice; leave them.
- No PowerShell was edited; no parser/ASCII validation step required. The harness was run, not modified.

status: Canonical Protocol Calibration Spot-Check v1 completed 2026-05-28. 3 live NBA runs through the canonical pipeline; cognitiveProtocol shape healthy end-to-end (scalar perceive, single discern.stress, no interrogate.stress / discern.test, populated probe, clean protocolView); runtime quality gates correctly silent on monitor postures; calibration delta directional only at n=3. v3 stamp is now safe. next: artifact v3 stamp + legacy block persistence decision. jera-workspace-skills untouched. COMMITTED+PUSHED: dai-vault afa319d (docs(sports): add canonical protocol calibration spot-check). No dai commit (no code changed).

## addendum: Artifact v3 Stamp v1 (2026-05-28)

Code slice (TDD, .NET + small Angular label touch). Migration plan Q6 + Q3 in one commit: stamp `sports_decision_artifact_v3` on every record from this slice forward, drop the legacy `CognitivePhases` block on success records, and recognize v3 on the dev artifact label. No FastAPI prompt, Pydantic, CognitiveProtocolBuilder mapping, confidence rule, posture behavior, Tool Gateway behavior, DB schema, MCP, pgvector, Azure Functions, Kubernetes, or secrets change. Pre-launch posture honored: no destructive purge of historical records.

### naming and skills gate

Skills used (jera pack, local, read-only):
- `dai-grill-with-vault` -- read migration plan Q6 + Q3, the spot-check note, `CognitiveProtocol.cs`, `SportsComposer.cs`, `AgentRunContracts.cs`, the artifact endpoint in `AgentRunsController.cs:282-314`, `ProtocolVocabularyMapper`, and `apps/sports-app/.../dev-artifact-review.component.ts:126-131` before naming/coding. Surfaced two semantic choices that needed explicit decisions: (1) failure-path version stamp, (2) what `null CognitivePhases` means for the read-side projection.
- `dai-token-tight` -- reporting density.
- `dai-agent-handoff` -- transfer notes.

Skills used (Claude built-ins):
- `superpowers:test-driven-development` -- RED on renamed v3 assertions + the new `compose_drops_legacy_cognitive_phases_on_v3_success_runs` test + the v3 controller integration test, then GREEN after the constant + composer changes. One genuine RED also surfaced an existing test (`compose_passes_cognitive_phases_from_analyzer`) that was incompatible with the new v3 doctrine -- it was retargeted in the same RED cycle (now `compose_drops_legacy_cognitive_phases_and_carries_canonical_protocol_through`).
- `superpowers:verification-before-completion` -- full dotnet suite + named-test confirmation; ASCII check on every changed `.cs` and the Angular `.ts` line.
- `superpowers:planning` / `superpowers:writing-plans` -- consulted; migration plan Q6 already specifies the sequence.

Skill weakness / sharpening recommendations carried forward:
1. The migration-step pattern "add version constant -> stamp success + failure -> drop legacy block on success -> tiny dev-page label tweak -> retain prior-version tests as backward-compat proofs" has now repeated four times (Stress Collapse, Perceive Scalar Collapse, Spot-Check, v3 Stamp). A dedicated `dai-implement-with-vault` skill that codifies this pattern would beat the interactive grill template.
2. The harness markdown reporter (the `interrogate.stress` / `discern.test` "not recorded" columns) remains the smallest open follow-up. Not addressed in this slice (scope discipline).
3. `jera-workspace-skills` left untouched (repo boundary rule honored, no approval to edit).

Naming decisions (gate item documented):
- **`ArtifactVersions.SportsDecisionArtifactV3 = "sports_decision_artifact_v3"`** -- mirrors V2 precedent; matches the migration plan Q6 vocabulary; no new naming family.
- **Failure-path version stamp choice: V3 uniformly.** Rationale: the version is an **era marker** of the pipeline that produced the record, not a shape marker of success vs failure. A failed run produced by the post-migration pipeline is still "from the v3 era." Success vs failure is differentiated by the presence of `CognitiveProtocol` (non-null on success, null on failure), not by `ArtifactVersion`. Documented in the `ArtifactVersions` class comment and on `AgentRunExecutionResult.ArtifactVersion` so a future reader does not re-litigate.
- **Legacy `CognitivePhases` on v3 success records: dropped (null).** Rationale: the read-side projection (`ProtocolVocabularyMapper.Project`) already prefers canonical when present, and the canonical `CognitiveProtocol` is now produced for every success record. Persisting both was justified during the V2 dual-emit window; with the canonical analyzer prompt shipped and the spot-check confirming the shape is healthy, the legacy block is redundant going forward. Old v1 records (which have `CognitivePhases` only and no canonical block) keep their persisted shape and remain readable via `ProjectFromLegacy`.
- **Legacy `CognitivePhases` on v3 failure records: null** -- analyze failed before any phase output existed; no change in semantics from the prior V2 failure path here.
- **Angular `/dev/artifacts` label**: added the V3 branch `'v3 sports_decision_artifact'`. V2 label retained; V1 (null) label retained. The dev page's source badge and rendering logic are unchanged because they already prefer `cognitiveProtocol` first.
- **No destructive purge** script in this slice -- pre-launch dev runs are disposable but were left readable. The user explicitly approved disposability "if needed"; this slice did not need it.

### files changed

- `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocol.cs` -- added `SportsDecisionArtifactV3` constant; expanded the `ArtifactVersions` doc comment to spell out the era-marker semantics (V1 null, V2 dual-emit, V3 canonical-authoritative; failure paths also stamp V3).
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsComposer.cs` -- success path: stamp V3 + null `CognitivePhases` (canonical `CognitiveProtocol` is the persisted authoritative shape); failure path: stamp V3 uniformly + null `CognitivePhases` + null `CognitiveProtocol` (era marker, not shape marker). Doc comments updated.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs` -- comment refresh on `AgentRunExecutionResult.CognitivePhases`, `AgentRunExecutionResult.ArtifactVersion`, `AgentRunExecutionResult.CognitiveProtocol`, and `AgentRunArtifactDto.CognitiveProtocol` to reflect v3 semantics (legacy block null on v3, canonical block authoritative on v2+v3).
- `dai/apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.ts` -- one-line addition: `if (version === 'sports_decision_artifact_v3') return 'v3 sports_decision_artifact'`. No HTML change; the existing source-badge logic continues to work because it already prefers the canonical block.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsComposerTests.cs` -- 4 renames + 1 new test:
  - `compose_stamps_v2_artifact_version_on_successful_runs` -> `compose_stamps_v3_artifact_version_on_successful_runs`.
  - new `compose_drops_legacy_cognitive_phases_on_v3_success_runs` -- asserts `CognitivePhases is null` and `CognitiveProtocol is not null` on a v3 success result.
  - `compose_canonical_protocol_is_null_when_analyzer_did_not_emit_phases` -- updated to assert V3 + null `CognitivePhases` + null `CognitiveProtocol`.
  - `compose_failed_run_stamps_v2_artifact_version_but_leaves_protocol_null` -> `compose_failed_run_stamps_v3_artifact_version_uniformly`, asserting V3 + null `CognitiveProtocol` + null `CognitivePhases`.
  - `compose_persists_probe_on_v2_when_market_grounded_but_sharp_public_missing` -> `compose_persists_probe_on_v3_...` (era-name only; behavior unchanged).
  - `compose_passes_cognitive_phases_from_analyzer` (the pre-existing test that asserted the legacy block passed through) -> `compose_drops_legacy_cognitive_phases_and_carries_canonical_protocol_through` -- now asserts the new v3 doctrine.
- `dai/platform/dotnet/DevCore.Api.Tests/Integration/AgentRunsControllerTests.cs` -- one new integration test (`artifact_endpoint_surfaces_artifact_version_and_canonical_protocol_when_v3_present`) and one new fixture (`BuildInspectableArtifactV3`). The pre-existing v2 test (`artifact_endpoint_surfaces_artifact_version_and_canonical_protocol_when_v2_present`) and v1 test (`artifact_endpoint_returns_null_artifact_version_for_v1_records`) are **deliberately kept unchanged** -- they are the backward-compatibility proofs that old persisted records of either prior era still read back cleanly through the v3-era controller and projection.
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

No FastAPI / Python change. No DB migration. No `dai-vault` calibration file change. No `jera-workspace-skills` change.

### version behavior

- **V3 success record** -- new persisted shape from this slice forward: `ArtifactVersion = "sports_decision_artifact_v3"`, `CognitiveProtocol != null`, `CognitivePhases = null`. All deliver-layer extracts (`Posture`, `CounterCase`, `WatchFor`, `WhatWouldChangeTheRead`), `Confidence`, `EvidenceRichness`, `ArtifactQualityWarnings`, `SignalAvailability`, `SignalFollowUps`, and pipeline metadata are unchanged.
- **V3 failure record** -- new persisted failure shape: `ArtifactVersion = "sports_decision_artifact_v3"`, `CognitiveProtocol = null`, `CognitivePhases = null`. Publishability remains `NotPublishable`; degradation notes and the failed analyze step are recorded as before.

### compatibility behavior

- **V1 records** (legacy, predate the canonical block): `ArtifactVersion = null`, `CognitivePhases` populated, `CognitiveProtocol = null`. Continue to deserialize cleanly. `ProtocolVocabularyMapper.Project` routes them through `ProjectFromLegacy` and surfaces the canonical stress via `Discern.Stress.Canonical` (with the vestigial legacy slots null) per Stress Collapse v1's view rules. The Angular dev page renders them as "v1 (legacy)".
- **V2 records** (dual-emit window): both `CognitivePhases` and `CognitiveProtocol` populated. Continue to deserialize cleanly. `ProtocolVocabularyMapper.Project` prefers canonical (`ProjectFromCanonical`). The Angular dev page renders them as "v2 sports_decision_artifact".
- **V3 records** (this slice): `CognitiveProtocol` only. The projection prefers canonical (same code path as v2). The Angular dev page renders them as "v3 sports_decision_artifact".
- Reader contract on the artifact endpoint is byte-additive: existing v2 consumers see no shape change for v2 records; v3 records expose `CognitivePhases = null` on the artifact DTO, which any consumer was already required to handle since the DTO has always declared it nullable. The customer-facing `AgentRunResultDto` is unchanged.

### test results

`dotnet test`: 277 passed, 0 failed (was 275; +2 net after one test rename + one new composer test + one new integration test). RED first on the renamed v3 assertions and the new tests; one additional RED cycle when the pre-existing legacy-pass-through test surfaced and was retargeted to the new doctrine. Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:617` is unrelated.

`pytest`: not run -- no Python file changed this slice.

### risks

- v3 records persist `CognitivePhases = null`. Any external consumer reading the persisted block directly (bypassing the artifact endpoint and the `ProtocolVocabularyMapper`) would see null where they previously got data. No in-repo consumer takes that path; the artifact endpoint, the read-side projection, and the Angular dev page all prefer the canonical block. Severity: low.
- The failure-path version stamp choice (V3 uniformly) was deliberate. A future reader who expects the version to mark "v3 = successful canonical record" would be wrong. The decision is documented in code and here; the differentiator is `CognitiveProtocol` presence. Severity: low.
- One existing test had to flip semantics (`compose_passes_cognitive_phases_from_analyzer` -> retargeted). It surfaced from the suite RED cycle and was renamed to assert the new doctrine; no orphan assertions remain. Severity: low.
- The "v3 stamp on failure" semantic is also documented in the `ArtifactVersions` and `AgentRunExecutionResult.ArtifactVersion` doc comments so the choice is visible at the read site. Severity: low.
- No data migration; old persisted records keep their stored shape and remain readable. Severity: low.

### next recommended slice

Plan step 7 alias removal (small, safe given v3 is now stamped and the canonical prompt is the only producer):
1. Pydantic alias removal: drop the `phases` validation alias on `SportsAnalysisResponse`, drop the `phases` read-property, drop the `SportsCognitivePhases` module alias in `app/models/sports.py`. The analyzer no longer emits legacy names; the wire DTO no longer needs to accept them.
2. .NET wire DTO alias trim: drop the legacy `Phases` JSON property + the `Test` / cross-block `InterrogatePhaseWire.Stress` aliases + the perceive `StringOrStringArrayJsonConverter` (the converter is only useful while legacy array payloads exist in flight; with the canonical prompt the only producer and v3 being scalar, the converter becomes inert).
3. Downstream .NET contract record rename: `SportsCognitivePhases` -> `SportsCognitiveProtocol` across `SportAnalysisContracts.cs`, `CognitiveProtocolBuilder.cs`, `ProtocolVocabularyMapper.cs`. Naming asymmetry from the prior slices closes.

Parallel candidate: teach `run-artifact-calibration.ps1` to read the canonical `cognitiveProtocol` block first to close the harness markdown reporter divergence (read columns from canonical, drop the retired `interrogate.stress` / `discern.test` columns).

### Claude <-> Codex transfer notes

- Repos in play: `dai` (6 files: 4 .NET runtime/tests + Angular 1 line + 1 contract comment) and `dai-vault` (this addendum). `jera-workspace-skills` untouched.
- Re-verify anywhere: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 277 passing. No stack/DB/network needed (composer/projection logic + WebApplicationFactory integration are pure).
- Stack state at slice close: Docker Desktop still running, `devcore-sql` up, FastAPI on `:8000` still up in its window; the .NET API on `:5007` was stopped this slice (its build output lock blocked the test rebuild) -- restart with `scripts/start-platform-api.ps1` if continuing. Or stop the whole stack via `scripts/dev/sports/stop-sports-dev.ps1`.
- Doctrine: ArtifactVersion is an era marker. Success vs failure is differentiated by `CognitiveProtocol` presence on the record, not by the version string. Do NOT add a "v3 failure" variant constant; do NOT decide failure by the version field.
- Reading old records: the artifact endpoint + `ProtocolVocabularyMapper` already handle V1/V2/V3 cleanly. Do NOT rewrite historical records.
- Pre-existing untracked `dai-vault` calibration files under `04 Products/sports-v1/calibration/` are NOT from this slice; leave them. Pre-existing untracked `dai/scripts/dev/sports/purge-dev-agent-runs.ps1` is NOT from this slice; leave it.
- No PowerShell changed this slice, so no ASCII/parser-validation step was required beyond ASCII checks on the changed `.cs` and `.ts` files.

status: Artifact v3 Stamp v1 implemented 2026-05-28. SportsDecisionArtifactV3 added; SportsComposer stamps V3 on success AND failure paths (era marker, not shape marker); legacy CognitivePhases dropped on v3 success records; CognitiveProtocol authoritative on v2+v3; v1 (legacy) records remain readable; Angular dev label recognizes v3. dotnet 277 (+2). pytest unchanged (no Python edit). next: plan step 7 alias removal + downstream record rename. jera-workspace-skills untouched. COMMITTED+PUSHED: dai 185eac3 (feat(protocol): stamp canonical sports artifacts as v3), dai-vault 2f9d220 (docs(protocol): document sports artifact v3 stamp).

## addendum: Canonical Protocol Alias Removal and Contract Rename v1 (2026-05-28)

Code slice (TDD, .NET + FastAPI + small Angular type follow-up). Migration plan step 7 plus the downstream contract record rename, in one commit. The active runtime is canonical-only after this slice: every legacy alias surface (Pydantic `validation_alias` / `populate_by_name`, `phases` block key, `.phases` response property, `SportsCognitivePhases` module alias, .NET wire field aliases for the eight legacy micro-action names, the cross-block stress fallback, the perceive list-to-scalar converter, the runtime contract `CognitivePhases` field, the `ProjectFromLegacy` projection path, the `StringOrStringArrayJsonConverter` file) was retired. Old persisted v1/v2 dev records no longer project; they are disposable per the approved pre-launch direction. No destructive purge was executed this slice. No confidence-rule, posture-behavior, Tool-Gateway, DB-schema, MCP, pgvector, Azure-Functions, Kubernetes, or secrets change. v3 artifact-version behavior is unchanged from the prior slice.

### naming and skills gate

Skills used (jera pack, local, read-only):
- `dai-grill-with-vault` -- read migration plan step 7 + 8, the vocabulary map, every alias surface (Pydantic models + parser + tests, `SportAnalysisContracts.cs` + `SportsAnalysisWire.cs` + `StringOrStringArrayJsonConverter.cs` + `CognitiveProtocolBuilder.cs` + `ProtocolVocabularyMapper.cs` + `SportsComposer.cs` + `AgentRunContracts.cs` + `AgentRunsController.cs`, the Angular dev-page DTO + component) before naming/coding. Locked the canonical-only contract in one pass to avoid mid-slice renames.
- `dai-token-tight` -- reporting density.
- `dai-agent-handoff` -- transfer notes shape.

Skills used (Claude built-ins):
- `superpowers:test-driven-development` -- RED on the new canonical-only Pydantic field-name tests + the new canonical-only wire tests, then GREEN after the lockstep renames; the broad RED also surfaced legacy-tolerance tests that had to be retargeted to canonical-only doctrine.
- `superpowers:verification-before-completion` -- full pytest + full dotnet + ASCII check on changed `.cs` / `.py` / `.ts` files; the runtime build was verified clean before touching the tests.
- `superpowers:systematic-debugging` -- used twice: once for the SportsComposerTests `Assert.Same(phases, ...CognitivePhases)` pass-through assertion that surfaced after the v3 doctrine flipped (retargeted), and once for the wire test that asserted "legacy list[str] yields null" but actually threw because the converter was removed (retargeted as an explicit Throws assertion plus a sibling "yields null micro-actions" test that doesn't include the list[str] perceive shape).
- `superpowers:planning` / `superpowers:writing-plans` -- consulted; the migration plan already specifies step 7. Locked the sequence in one slice instead of staging.

Skills weakness / sharpening recommendations carried forward:
1. The migration-step pattern "add canonical surface -> bridge with alias scaffold -> flip prompt -> stamp era constant -> drop legacy surfaces" has now repeated five times. A `dai-implement-with-vault` skill codifying this would beat the interactive grill template.
2. Wide test-file sweeps for symbol/field renames are still painful: a small `dai-rename-protocol` script (the canonical->legacy substitution map plus the drop list of retired tests) would shorten future renames significantly. Not in scope this slice.
3. `jera-workspace-skills` left untouched (repo boundary rule honored, no approval to edit).

Naming review result (gate item documented):
- **DevCore.AiClient renames (canonical-only):**
  - `SportsCognitivePhases` -> `SportsCognitiveProtocol`
  - `SportsPerceivePhase` -> `SportsPerceiveProtocol` (Detect / Frame / Aim unchanged)
  - `SportsInterrogatePhase` -> `SportsInterrogateProtocol`; fields `Balance` -> `Question`, `Reframe` -> `Verify`
  - `SportsDiscernPhase` -> `SportsDiscernProtocol`; fields `Listen` -> `Contrast`, `Filter` -> `Weigh`, Stress unchanged
  - `SportsDecidePhase` -> `SportsDecideProtocol`; fields `Calibrate` -> `Justify`, `Posture` -> `Position`, `Voice` -> `Resolve`
  - `SportsAnalysisResponse.Phases` -> `.Protocol` with `[JsonPropertyName("protocol")]`
- **Pydantic renames (canonical-only):** matching class + field renames in `app/models/sports.py`; `populate_by_name`, `validation_alias=AliasChoices(...)`, the `.phases` `@property`, `PerceiveScalar` + `_coerce_str_or_str_list`, and the `SportsCognitivePhases` module alias all dropped.
- **Builder rename:** `CognitiveProtocolBuilder.FromLegacy` -> `FromAnalyzerProtocol` -- the method now consumes the analyzer's canonical `SportsCognitiveProtocol` and produces the runtime `CognitiveProtocol`. The two records live in different namespaces (DevCore.AiClient wire vs DevCore.Api.AgentRuns persistence) and that separation stays intentional; the cross-namespace translation, the deterministic Probe injection, and the Synthesize platform-operational constants are still its job.
- **Wire DTO renames:** `InterrogatePhaseWire` -> `InterrogateProtocolWire`, `DiscernPhaseWire` -> `DiscernProtocolWire`, `DecidePhaseWire` -> `DecideProtocolWire`, `PerceivePhaseWire` -> `PerceiveProtocolWire`. All legacy JSON property name aliases removed. `CognitiveProtocolWire.Phases` JSON property removed. Cross-block stress fallback in `CognitiveProtocolWire.ToModel()` removed. `[JsonConverter(StringOrStringArrayJsonConverter)]` removed from perceive properties.
- **Converter file:** `StringOrStringArrayJsonConverter.cs` DELETED (no remaining consumer; the canonical prompt is the only producer of perceive payloads, and the canonical contract is scalar).
- **Runtime contract field:** `AgentRunExecutionResult.CognitivePhases` removed; `AgentRunArtifactDto.CognitivePhases` removed. The single cognitive persistence surface is `CognitiveProtocol`.
- **Mapper:** `ProtocolVocabularyMapper.ProjectFromLegacy` removed. `Project()` returns null when `CognitiveProtocol` is absent. The `DiscernStressProtocolView` shape retains its two legacy slots (Angular DTO contract unchanged); they are vestigial and always null.
- **Parser:** `_parse_phases` -> `_parse_protocol`. Canonical-only field reads. Cross-block stress fallback, perceive list-to-scalar coercion, and the `_pick(canonical, legacy)` helper all dropped. Reads only the `protocol` block key.
- **Prompt:** the analyzer's `_JSON_SHAPE` flipped from the `phases` block key to the `protocol` block key, and the per-field instruction cross-references flipped from `phases.<protocol>.<field>` to `protocol.<protocol>.<field>`. Only the block-key rename and the cross-reference rename were made; the per-field instruction content, the prohibited-phrase lists, the no-fabrication guardrails, and the field semantics are unchanged.
- **Angular tiny follow-up:** `cognitivePhases` removed from `AgentRunArtifactDto` (TS type); the dev page's `phaseCards` computed signal now passes `null` (the canonical projection drives all rendering through `protocolBlocks`). No HTML or visual change required because the page already preferred the canonical block and the `phaseCards` rendering only fires when the legacy block was non-null.

### compatibility behavior

- Active runtime: canonical-only. Every cognitive surface (prompt, parser, wire DTO, runtime contract, builder, mapper, persistence, dev page) uses canonical names and the canonical block key.
- Pre-canonical persisted records (v1, v2): no longer project. v1 (`CognitiveProtocol = null`, only the retired legacy block populated) and v2 (both blocks populated -- the canonical block still works for v2 because `CognitiveProtocol` was added at v2) records that have only the legacy block produce a `null ProtocolView`. v2 records that DO have `CognitiveProtocol` populated still read fine (the canonical path is unchanged).
- Pre-v3 dev records that block tests or queries can be purged via the untracked dev script `dai/scripts/dev/sports/purge-dev-agent-runs.ps1`. **Not executed this slice.** The user explicitly carried it as out-of-scope. Operators on dev machines can run it locally if needed; the script is not committed.
- No data migration. No DB schema change.
- The artifact endpoint's JSON output drops the `cognitivePhases` property (was always nullable on the DTO and is null on every v3 record anyway). Existing Angular consumers were already handling the field as nullable; the dev page reference was updated this slice.

### files changed

dai:
- `dai/services/agent-service/app/models/sports.py` -- canonical-only Pydantic models. Class renames + field renames; `populate_by_name`/`validation_alias`/`AliasChoices`/`BeforeValidator`/`ConfigDict` imports and `.phases` property and `SportsCognitivePhases` alias and `PerceiveScalar`/`_coerce_str_or_str_list` all dropped.
- `dai/services/agent-service/app/services/sports_analyzer.py` -- prompt block key flipped to `protocol`; per-field cross-references flipped; parser rewritten as `_parse_protocol` (canonical-only, no `_pick` / `_s_or_list` / cross-block stress fallback); `_parse_response` reads only the `protocol` block; `counter_case` source now `protocol.interrogate.question`, `watch_for` source `protocol.discern.stress`, posture fallback `protocol.decide.position`.
- `dai/services/agent-service/tests/test_sports_analyzer.py` -- canonical-only test fixtures (`_make_full_protocol`); legacy alias scaffold section, Stress Collapse boundary section, Perceive Scalar Collapse boundary section, and Analyzer Protocol Class Rename back-compat-property tests all removed; new canonical-only block added with canonical-shape tests, structural model-fields guards, parser ignores-legacy tests, and prompt canonical-block assertions.
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs` -- canonical analyzer contract; class + field renames; `SportsAnalysisResponse.Protocol` with `[JsonPropertyName("protocol")]`.
- `dai/platform/dotnet/DevCore.AiClient/SportsAnalysisWire.cs` -- canonical-only wire DTO; only `protocol` block key accepted; only canonical micro-action JSON property names accepted; cross-block stress fallback removed; perceive converter removed.
- `dai/platform/dotnet/DevCore.AiClient/StringOrStringArrayJsonConverter.cs` -- **DELETED** (no remaining consumer).
- `dai/platform/dotnet/DevCore.AiClient/FastApiClient.cs` -- comment refresh to reflect canonical-only behavior.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolBuilder.cs` -- `FromLegacy` -> `FromAnalyzerProtocol`; signature flipped to take `SportsCognitiveProtocol`; canonical pass-through.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/ProtocolVocabularyMapper.cs` -- `ProjectFromLegacy` dropped; `Project` returns null when `CognitiveProtocol` is null; canonical projection path unchanged.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsComposer.cs` -- reads `analyzerOutput.Protocol`; calls `FromAnalyzerProtocol`; the `CognitivePhases: null` ctor argument is gone with the field removal.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs` -- `AgentRunExecutionResult.CognitivePhases` removed; `AgentRunArtifactDto.CognitivePhases` removed; comment refresh.
- `dai/platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocol.cs` -- comment refresh.
- `dai/platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs` -- DTO construction no longer passes `artifact.CognitivePhases`.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsAnalysisWireTests.cs` -- rewritten canonical-only: canonical happy path, null protocol, legacy-block-key ignored, legacy-field-names-inside-canonical-block yields null micro-actions, legacy perceive list[str] now throws at the boundary, structural guard that the wire has no `Phases` property.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/ProtocolVocabularyMapperTests.cs` -- rewritten canonical-only: null result, null CognitiveProtocol, canonical field mapping, canonical stress in Canonical slot with legacy slots null, scalar-to-array round-trip, Probe preservation, every validated position round-trips exactly, Synthesize platform-operational guard.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/CognitiveProtocolBuilderTests.cs` -- mechanical: type names and method name renamed; constructor args use canonical names; assertions unchanged in semantics.
- `dai/platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsComposerTests.cs` -- mechanical type renames; `Assert.Null(..CognitivePhases)` lines dropped; the `MakePhases` fixture is canonical-typed.
- `dai/platform/dotnet/DevCore.Api.Tests/Integration/AgentRunsControllerTests.cs` -- mechanical type renames; v1-record-projects-cleanly test retargeted to "protocolView is null when CognitiveProtocol absent"; `dto.CognitivePhases` assertions dropped; `BuildInspectableArtifact` fixture updated to feed `FromAnalyzerProtocol` for the inspectable path; `nameof(AgentRunArtifactDto.CognitivePhases)` line dropped from the recent-runs DTO surface check.
- `dai/apps/sports-app/src/app/core/models/agent-run.model.ts` -- `cognitivePhases` field dropped from `AgentRunArtifactDto`; comment refresh.
- `dai/apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.ts` -- `phaseCards` computed now passes null; comment refresh.

dai-vault:
- `dai-vault/06 Execution/handoffs/current-slice.md` (this addendum).

jera-workspace-skills: untouched (read-only).

### test results

- pytest (`tests/test_sports_analyzer.py`): 100 passed (was 117 from the v3 stamp slice; -17 net after dropping the alias-scaffold + Stress-Collapse boundary + Perceive-Scalar boundary + Analyzer-Protocol-Class-Rename back-compat sections). All canonical-only assertions GREEN.
- dotnet test (full suite): 256 passed, 0 failed (was 277 from the v3 stamp slice; -21 net after dropping the legacy-acceptance + ProjectFromLegacy + StringOrStringArrayJsonConverter + CognitivePhases-DTO sections plus a few mechanical consolidations). Pre-existing xUnit2013 warning at `AgentRunsControllerTests.cs:588` is unrelated.

### risks

- pre-canonical dev records (v1, v2 with only the legacy block) no longer project. The user explicitly approved disposability. The untracked `purge-dev-agent-runs.ps1` is the operator's path for cleaning a local dev db; it was NOT executed this slice. Severity: low (dev-only).
- the analyzer prompt's `protocol` block key is a fresh change; the model has not been live-verified against the new prompt this slice. A short calibration spot-check on the next live batch is prudent before any threshold change; confidence math is unchanged so the risk is purely texture/shape. Severity: low-med (texture), low (correctness).
- the wide rename touches many test fixtures. The full dotnet suite + pytest suite are GREEN, but the change reaches into integration tests via `BuildInspectableArtifact` which now feeds the canonical block through the builder. Severity: low (compile-checked + suite-checked).
- `DiscernStressProtocolView` retains its two vestigial legacy slots so the Angular DTO contract is unchanged. A future slice may drop them once the dev page is confirmed not to reference them; not in scope here. Severity: low.

### next recommended slice

Calibration spot-check on the new canonical-only analyzer prompt (the `protocol` block key flip is fresh). Run 5-8 live games through the canonical pipeline and confirm the model honors the new block key cleanly. After that, the natural next surfaces are:
1. Drop the vestigial `DiscernStressProtocolView.LegacyInterrogateStress` / `.LegacyDiscernTest` slots (the Angular dev page renders only the `Canonical` slot today).
2. Update the harness markdown reporter (`run-artifact-calibration.ps1`) to read the canonical `cognitiveProtocol` block first, closing the long-standing reporter divergence on retired `interrogate.stress` / `discern.test` columns.
3. Update `dai-vault/02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md` to mark legacy names retired-from-runtime (migration plan step 8).

### Claude <-> Codex transfer notes

- Repos in play: `dai` (Pydantic 2 files; FastAPI tests; .NET 5 production files + 5 test files; Angular 2 files) and `dai-vault` (this addendum). `jera-workspace-skills` untouched.
- Re-verify anywhere: `pytest tests/test_sports_analyzer.py` -> 100 passing; `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj` -> 256 passing. No stack/DB/network needed.
- Canonical-only doctrine: the active runtime accepts only canonical field names and the canonical `protocol` block key. Do NOT reintroduce a legacy alias scaffold for any reason short of a calibration regression that pinpoints the prompt-content change as the cause; the alias surfaces were retired deliberately.
- Pre-canonical dev records (v1, v2-only-with-legacy-block) no longer project. The dev purge command is `pwsh scripts/dev/sports/purge-dev-agent-runs.ps1` (untracked) -- run it on a dev box when needed; the script is intentionally NOT committed.
- The builder lives in two namespaces by design: `DevCore.AiClient.SportsCognitiveProtocol` is the wire shape; `DevCore.Api.AgentRuns.CognitiveProtocol` is the persistence shape. Their fields share canonical names but the records are separate types. The builder's job is the cross-namespace translation plus deterministic Probe + Synthesize.
- Pre-existing untracked `dai/scripts/dev/sports/purge-dev-agent-runs.ps1` is NOT from this slice. Pre-existing untracked `dai-vault` calibration files under `04 Products/sports-v1/calibration/` are NOT from this slice either. Leave both.
- No PowerShell was edited; no parser/ASCII validation step beyond ASCII checks on changed `.cs` / `.py` / `.ts` files.

status: Canonical Protocol Alias Removal and Contract Rename v1 implemented 2026-05-28. Active runtime is canonical-only. Legacy aliases retired across Pydantic (validation_alias, populate_by_name, `.phases` property, SportsCognitivePhases module alias, PerceiveScalar/BeforeValidator), parser (cross-block stress fallback, legacy field-name reads, legacy block key), .NET wire DTO (legacy field-name JSON aliases, legacy `Phases` block key, cross-block stress fallback), .NET runtime contract (CognitivePhases field on AgentRunExecutionResult + AgentRunArtifactDto), mapper (ProjectFromLegacy), and file system (StringOrStringArrayJsonConverter.cs deleted). Builder renamed FromLegacy -> FromAnalyzerProtocol. Analyzer prompt flipped to `protocol` block key with canonical per-field cross-references. v3 artifact-version behavior unchanged. Angular TS type pruned. pytest 100 (-17). dotnet 256 (-21). next: calibration spot-check on the canonical-only prompt + retire vestigial legacy stress view slots + update calibration harness reporter + mark legacy names retired-from-runtime in the vocabulary map. jera-workspace-skills untouched. COMMITTED+PUSHED: dai cc5c924 (feat(protocol): remove legacy analyzer aliases), dai-vault a098ae9 (docs(protocol): document canonical alias removal).

## addendum: .NET Test Verification Hardening v1 (2026-05-28)

Dev tooling slice. No runtime behavior changed. Adds a safe repo-level wrapper for `DevCore.Api.Tests` so agents do not run the unstable full .NET test command blindly when it hangs or returns `Build FAILED` with `0 Error(s)`.

### naming and skills gate

Naming result:
- script name: `scripts/dev/dotnet/test-devcore-api-safe.ps1`
- scope: dev-only verification for `platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj`
- no runtime/domain naming touched
- no FastAPI prompts, CognitiveProtocol behavior, Tool Gateway behavior, confidence rules, DB schema, Angular behavior, MCP, pgvector, Azure Functions, Kubernetes, or secrets touched

Skills used:
- `dai-grill-with-vault` -- read existing dev/test docs and current handoff context before placing the script/docs.
- `dai-token-tight` -- concise status and final report.
- `dai-agent-handoff` -- this addendum shape.
- `superpowers:planning` / `superpowers:writing-plans` -- locked the small sequence before edits.
- `superpowers:test-driven-development` -- script behavior treated as testable: parser validation, targeted mode, full mode.
- `superpowers:systematic-debugging` -- preserves the discovered failure mode: plain full command can hang or fail without useful output; targeted-first is the diagnostic route.
- `superpowers:verification-before-completion` -- parser check plus targeted and full script execution before close.

### files changed

dai:
- `scripts/dev/dotnet/test-devcore-api-safe.ps1` -- new safe runner. Default: targeted tests first, then full suite. Supports `-Targeted`, `-Full`, and prints each command before execution. Every dotnet test command uses `/m:1 /nr:false`.
- `scripts/dev/dotnet/README.md` -- documents the safe runner and when to use targeted-first.
- `platform/dotnet/DevCore.Api.Tests/README.md` -- points agents to the safe runner and updates manual commands to use `-v minimal /m:1 /nr:false`.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:
- untouched.

### verification results

- PowerShell parser validation for `scripts/dev/dotnet/test-devcore-api-safe.ps1`: 0 parser errors; ASCII check passed.
- `powershell -ExecutionPolicy Bypass -File .\scripts\dev\dotnet\test-devcore-api-safe.ps1 -Targeted`: 64 passed, 0 failed. Command printed before execution; used `/m:1 /nr:false`.
- `powershell -ExecutionPolicy Bypass -File .\scripts\dev\dotnet\test-devcore-api-safe.ps1 -Full`: 256 passed, 0 failed. Command printed before execution; used `/m:1 /nr:false`.

### skill sharpening recommendation

DAI verification skills should use `test-devcore-api-safe.ps1` or `/m:1 /nr:false` when full dotnet test hangs or returns `Build FAILED` with `0 Error(s)`.

No local skill guidance was edited because the user did not approve `jera-workspace-skills` changes for this slice.

status: .NET Test Verification Hardening v1 implemented 2026-05-28. Script and docs added. Parser/ASCII, targeted mode, and full mode verified clean. jera-workspace-skills untouched.

## addendum: Protocol Completion Representation v1 (2026-05-28)

Representation slice. No runtime behavior changed. Makes the seed-vs-completed distinction explicit in the type system so Interrogate never reads as a two-station macro.

### doctrine made explicit

- `AnalyzerProtocolSeed` (FastAPI `SportsAnalyzerProtocolSeed` + .NET wire `SportsAnalyzerProtocolSeed`) is **model-owned**: 11 cognitive station outputs from one analyze call.
- `CognitiveProtocol` (`.NET DevCore.Api.AgentRuns`) is **platform-completed**: the 11 model-emitted stations plus `interrogate.probe` (deterministic) and the Synthesize trio.
- 12 cognitive micro-actions = **11 model-emitted + 1 deterministic (`interrogate.probe`)**. `interrogate.probe` is completed by `CognitiveProtocolBuilder.BuildProbe` from signal follow-up records; the model never emits it.
- Synthesize (integrate, compose, deliver) is platform-owned and not among the 12.
- The completed artifact still contains `interrogate.question`, `interrogate.probe`, and `interrogate.verify`.

### naming review result

Hard rename approved (option 2). The model-emitted types now carry seed terminology in both languages; the completed artifact keeps `CognitiveProtocol`. The wire/prompt json block key stays `"protocol"` (only type names changed).

- FastAPI + .NET wire: `SportsCognitiveProtocol` -> `SportsAnalyzerProtocolSeed`, `SportsPerceiveProtocol` -> `SportsAnalyzerPerceiveSeed`, `SportsInterrogateProtocol` -> `SportsAnalyzerInterrogateSeed`, `SportsDiscernProtocol` -> `SportsAnalyzerDiscernSeed`, `SportsDecideProtocol` -> `SportsAnalyzerDecideSeed`.
- Completed .NET artifact unchanged: `CognitiveProtocol`, `PerceiveProtocol`, `InterrogateProtocol`, `DiscernProtocol`, `DecideProtocol`, `SynthesizeProtocol`.
- Builder method: `CognitiveProtocolBuilder.FromAnalyzerProtocol` -> `FromAnalyzerProtocolSeed`.

### files changed

dai:
- `services/agent-service/app/models/sports.py` -- seed class rename + seed docstrings.
- `services/agent-service/app/services/sports_analyzer.py` -- seed type rename (imports + usage).
- `services/agent-service/tests/test_sports_analyzer.py` -- seed rename + 3 new tests (seed parses from `protocol`; seed has question+verify not probe; prompt does not request probe).
- `platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs`, `SportsAnalysisWire.cs` -- seed record rename + seed docstrings.
- `platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocol.cs` -- fixed stale "probe remains null because no runtime source emits it" comment; completed/seed framing.
- `platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolBuilder.cs`, `SportsComposer.cs`, `AgentRunContracts.cs` -- method rename + completion-doctrine comments.
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/CognitiveProtocolBuilderTests.cs` -- rename + 3 new completed-shape tests; stale comment/name fixes.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolRegistryTests.cs` -- 3 new tests (each cognitive macro has 3 stations; interrogate has 3 incl. probe; probe is deterministic, no model call).
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/SportsComposerTests.cs`, `Integration/AgentRunsControllerTests.cs` -- seed rename.

dai-vault:
- `02 Platform/architecture/cognitive-factory/protocol-station-blueprint-v1.md`, `protocol-node-specs.md`, `protocol-vocabulary-map.md` -- corrected the 11/1/3 split and added seed-vs-completed terminology.
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### verification results

- pytest: 103 passed (was 100; +3).
- safe .NET runner targeted: 64 passed, 0 failed.
- safe .NET runner full: 262 passed, 0 failed (was 256; +6).
- No Angular change (the dev artifact surface already rendered Probe); Angular build not required.
- No confidence-rule, posture, Tool Gateway, schema, MCP, pgvector, Azure, or Kubernetes change.

status: Protocol Completion Representation v1 implemented 2026-05-28. Model-emitted seed renamed to AnalyzerProtocolSeed terminology in both languages; completed CognitiveProtocol unchanged; probe stays deterministic; json key `protocol` unchanged. pytest 103, dotnet 262. jera-workspace-skills untouched.

## addendum: Retire Vestigial Stress View Slots v1 (2026-05-29)

Cleanup slice. No runtime behavior changed. Removes the vestigial read-side stress wrapper so no endpoint or dev surface advertises retired stress sources.

### what was vestigial

The named legacy slots `LegacyInterrogateStress` / `LegacyDiscernTest` did not exist (already removed in Stress Collapse v1 / Canonical Alias Removal v1). The remaining vestige was the projection wrapper `DiscernStressProtocolView { Canonical }` (a holdover from when interrogate.stress and discern.test were separate stress sources) and the Angular "Canonical Stress" sub-label. With one canonical stress source, the wrapper implied multiplicity that no longer exists.

### change

- `DiscernProtocolView.Stress` collapsed from `DiscernStressProtocolView` to a plain `string?`, matching Weigh and Contrast. The `DiscernStressProtocolView` record was deleted (C# and the Angular DTO type).
- `ProtocolVocabularyMapper` now sets `Stress: protocol.Discern?.Stress` directly (no wrapper).
- Angular dev artifact page renders Stress as one normal "Stress" field; the `stressField` helper and the "Canonical Stress" sub-label were removed.
- The completed runtime contract (`CognitiveProtocol.Discern.Stress`), the canonical `discern.stress` station (`StationIds.DiscernStress`), confidence, posture, and the FastAPI prompt are untouched.
- Projection is request-time only and not persisted, so no migration; old dev runs are disposable pre-launch.

### naming review result

`DiscernProtocolView` now has three uniform `string?` fields (Weigh, Contrast, Stress). No new names introduced; the misleading nested wrapper removed. No surface advertises `interrogate.stress` or `discern.test` as active runtime fields.

### files changed

dai:
- `platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolView.cs` -- collapse Stress to string; delete DiscernStressProtocolView; comment update.
- `platform/dotnet/DevCore.Api/AgentRuns/ProtocolVocabularyMapper.cs` -- surface stress directly.
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/ProtocolVocabularyMapperTests.cs`, `Integration/AgentRunsControllerTests.cs` -- assert `Discern.Stress` as a string.
- `apps/sports-app/src/app/core/models/agent-run.model.ts` -- drop DiscernStressProtocolView type; stress is string.
- `apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.ts` -- render Stress as a plain field; remove stressField helper.

dai-vault:
- `02 Platform/architecture/cognitive-factory/protocol-vocabulary-map.md` -- read-side note that the stress wrapper is retired.
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### verification results

- safe .NET runner targeted: 64 passed, 0 failed.
- safe .NET runner full: 262 passed, 0 failed.
- Angular build: bundle generation complete, 0 errors (dev-artifact-review-component rebuilt).
- No Python change -> pytest not run.
- No confidence/posture/Tool-Gateway/schema/MCP/pgvector/Azure/Kubernetes/prompt change.

status: Retire Vestigial Stress View Slots v1 implemented 2026-05-29. DiscernStressProtocolView wrapper removed; read-side discern.stress is a single string in C# and Angular; canonical discern.stress station and runtime contract unchanged. dotnet 262, Angular build clean. jera-workspace-skills untouched.

## addendum: Per-Station Tool Gateway Policy v1 (2026-05-29)

Tool Gateway slice. The analyzer call is not split and the runtime pipeline is unchanged. Teaches the gateway to authorize canonical station-id callers (for future Protocol Node Runner work) while preserving current stage-sentinel behavior exactly.

### skills used

- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault (read code+vault before touching the security boundary), dai-token-tight, dai-agent-handoff. Pack not edited.
- superpowers: writing-plans (designed against real code before editing the gateway), verification-before-completion (all results below are fresh runs), test-driven-development (policy + gateway tests). systematic-debugging not needed.

### naming decisions

Accepted the candidates: `IProtocolToolAccessPolicy` / `ProtocolToolAccessPolicy`, with method `IsAllowed(ToolDefinition tool, string protocolNode)`. Placed in `DevCore.Api.Protocols` next to `ProtocolRegistryValidator` (the policy is the runtime dual of that test-only validator and reuses its `ApplicableStageSentinel`). No tool ids or station ids changed.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProtocolToolAccessPolicy.cs` -- NEW. The policy + interface + static `Default` built from `ProtocolRegistry.Default()`.
- `platform/dotnet/DevCore.Api/Tools/ToolGateway.cs` -- resolve `IProtocolToolAccessPolicy` from the injected IServiceProvider (fallback to `ProtocolToolAccessPolicy.Default`); replace the inline `AllowedProtocolNodes.Contains` check with `policy.IsAllowed`. Constructor signature unchanged.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- register `IProtocolToolAccessPolicy` singleton.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolToolAccessPolicyTests.cs` -- NEW. 8 unit tests.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayAnalyzeTests.cs` -- 1 new gateway test: a station-id caller (interrogate.question) routes the analyze tool through successfully.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### policy behavior

`IsAllowed(tool, node)`:
1. If `node` is a registered station id (in ProtocolRegistry): allow only if the station card lists the tool in AllowedTools AND the tool's AllowedProtocolNodes contains the exact station id OR the station's applicable stage sentinel (`ProtocolRegistryValidator.ApplicableStageSentinel`). A station whose card omits the tool fails closed.
2. Otherwise (a platform stage sentinel, or any other id): exact match against the tool's AllowedProtocolNodes -- identical to the v1 gateway. Unknown station-like ids are not in the registry, so they fall here and fail closed.

### compatibility behavior

- platform.reference / platform.retrieve / platform.analyze callers are unchanged (path 2 exact match). All existing allow and deny gateway tests pass unchanged.
- Today every runtime caller passes a stage sentinel, so the station-id path is dormant in production; it only affects future station-id callers.
- The gateway still throws ToolNotRegistered / ToolNotAllowedForProtocolNode with the same fields, and the denied/not_registered/handler_error telemetry outcomes are preserved (only the allow decision moved into the policy).
- The startup guard (ProtocolRegistryValidator) is unchanged and still green.

### test results

- safe .NET runner targeted: 64 passed, 0 failed.
- safe .NET runner full: 271 passed, 0 failed (was 262; +8 policy unit tests, +1 gateway station-id success test).
- No Python change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The gateway authorization decision was extracted to a policy that is a strict superset of the old behavior for non-station nodes and adds a fail-closed station-id path. Reuses the already-validated `ApplicableStageSentinel` logic. No tool/station id changes, no pipeline split, no prompt/confidence/posture/schema change. Reversible (revert the gateway line + remove the policy registration).

### next recommended slice

Protocol Node Runner v1 groundwork: introduce a caller that actually passes station ids through the gateway (e.g., a deterministic station such as a future memory-backed interrogate.probe), now that the policy authorizes them. Alternatively, promote `ProtocolRegistryValidator` from test-only/startup-guard to also back the policy's manifest so there is a single source.

### Claude/Codex transfer notes

- The policy is dormant for production callers; do not expect behavior change in current runs. To exercise it, pass a `ToolInvocationContext.ProtocolNode` equal to a `StationIds.*` value whose card lists the tool.
- The gateway resolves the policy from IServiceProvider with a `ProtocolToolAccessPolicy.Default` fallback, so test service graphs need no policy registration. Production registers it in `AddDaiToolGateway`.
- Do not split the analyzer call, do not make the model emit probe, do not change FastAPI prompt/confidence/posture in follow-ups unless explicitly scoped.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Per-Station Tool Gateway Policy v1 implemented 2026-05-29. ProtocolToolAccessPolicy added and wired into ToolGateway; station-id callers authorized via station card + applicable stage sentinel; stage-sentinel behavior and gateway telemetry preserved; analyzer call not split. dotnet 271 (targeted 64), no Python/Angular change. jera-workspace-skills untouched.

## addendum: Canonical Calibration Reporter v1 (2026-05-29)

Reporting/docs slice. No runtime behavior changed; no .NET, Python, or Angular change. The calibration harness was already migrated to canonical naming in the protocol-cleanup checkpoint; this slice verifies that, documents the canonical protocol surface, and makes the harness ASCII-only.

### inspection result

`scripts/dev/sports/run-artifact-calibration.ps1` already reads the canonical persisted `cognitiveProtocol` (7 reads) and contains zero retired tokens (no `cognitivePhases`, `interrogate.stress`, `discern.test`, `stress.canonical`, or legacy action names balance/reframe/listen/filter/calibrate/voice). The Cognitive Protocol Completeness and Cognitive Protocol Quality tables already use the 12 canonical micro-actions; the assessment flags already use canonical names (contrast_missing_with_external_signal, protocol_unsupported_claim). So tasks 2 and 3 were already satisfied by the checkpoint; no retired-field logic remained to remove.

### naming/reporting review result

No new identifiers introduced. Kept the canonical field names. The Synthesize trio (integrate, compose, deliver) is platform-owned and constant per run, so it is intentionally NOT added as per-run columns (item 4 "where useful" -> a documentation note is the useful form, not noise columns).

### files changed

dai:
- `scripts/dev/sports/run-artifact-calibration.ps1` -- added a canonical-only clarifying comment above the completeness table and a markdown note in the report (reads canonical cognitiveProtocol; interrogate.probe is platform-completed deterministic; synthesize is platform-owned and not a per-run column). Converted the 28 pre-existing non-ascii report glyphs (em-dash, middot) to ASCII "-" so the harness is ASCII-only; future reports use "-" for not-recorded cells. Historical exports are not regenerated.
- `scripts/dev/sports/README.md` -- documented that the protocol tables read canonical cognitiveProtocol, that synthesize is platform-owned, and that retired fields are not presented as active.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum. Historical calibration exports under `04 Products/sports-v1/calibration/` are preserved unchanged (item 5).

### reporter behavior

- Reads `entry.Artifact.cognitiveProtocol` (canonical persisted v3 surface). Records without it (older `cognitivePhases` exports, or failure records) render as not-recorded rows -- no retired field is read or presented as active.
- The two protocol tables show the 12 cognitive micro-actions. interrogate.probe is shown (platform-completed). Synthesize is documented as platform-owned, not a per-run column.

### compatibility behavior for old exports

- Historical calibration markdown and artifact JSON files in the vault are untouched (not rewritten).
- Old saved artifacts that carry `cognitivePhases` (pre-rename) simply render as not-recorded in the cognitive tables when re-read, which is the correct canonical-only behavior; they are not reinterpreted.
- The em-dash -> "-" change affects only future generated reports, not existing files.

### verification results

- PowerShell parser validation: 0 errors.
- ASCII validation: 0 non-ascii bytes (was 28 lines of em-dash/middot; now ASCII-only).
- Static canonical audit: reads cognitiveProtocol only; zero retired-field reads.
- Dry-run/export: the harness has a `-DryRun` switch, but it fetches upcoming games from the live platform API before the dry-run exit, so a real export requires the running API plus billable model calls. Not run offline; validated via parser + ASCII + static audit instead.
- No .NET/Python/Angular change -> those test suites not run.

### risks

Very low. Documentation + comment additions plus a cosmetic non-ascii-to-ascii conversion of report placeholder glyphs. No logic, field-read, confidence, posture, prompt, gateway, registry, schema, or Angular change. Historical exports preserved.

### next recommended slice

Either Protocol Node Runner v1 groundwork (a caller that passes station ids through the now-station-aware Tool Gateway), or a live calibration batch (run the harness against the running platform to confirm a real v3 report renders the canonical tables end-to-end) when the dev API is up.

### Claude/Codex transfer notes

- The harness reads canonical `cognitiveProtocol` from the live artifact endpoint; it does not read retired fields. Do not reintroduce a `cognitivePhases` fallback.
- To run a real calibration export: start the platform API, then `scripts/dev/sports/run-artifact-calibration.ps1 -Competition nba -Take 5` (each run is a billable model call; use `-DryRun` to list games without creating runs).
- Synthesize is platform-owned and constant; do not add it as per-run columns.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Canonical Calibration Reporter v1 implemented 2026-05-29. Harness verified canonical (reads cognitiveProtocol, zero retired tokens); added canonical/synthesize documentation note + comment; converted report glyphs to ASCII (parser 0 errors, 0 non-ascii). Historical exports preserved. No .NET/Python/Angular change. jera-workspace-skills untouched.

## addendum: Live Canonical v3 Calibration Batch v1 (2026-05-29)

Live verification slice. No code changed. Ran the real pipeline end-to-end against the running stack to confirm artifact v3, canonical CognitiveProtocol, deterministic probe, and the canonical reporter, all on freshly generated runs.

### stack / smoke

- devcore-sql: docker container up (port 1433). Outbound egress to OpenAI + odds/espn/mlb providers confirmed reachable.
- Started agent-service (pre-existing instance already on :8000) and a fresh .NET platform API (:5007) from current canonical source. Angular not needed.
- test-sports-dev.ps1 smoke: 5/5 passed (agent ping, direct analyze, stub-detection, unsupported-competition 400 gate, full chain via .NET created a completed run).
- Note: run-artifact-calibration.ps1 uses PowerShell 7 syntax (?. and ??) but lacks a `#requires -version 7` line, so it must be run with `pwsh` not Windows PowerShell 5.1. Not fixed here (no-code-change slice); flagged as a tiny follow-up.

### runs generated

8 MLB runs (2026-05-29 slate), all status completed:
068e433e, 098e433e, 0c8e433e, 0f8e433e, 158e433e, 178e433e, 198e433e, 1c8e433e. Plus the smoke full-chain run 028e433e (NFL).

### artifact + canonical findings

- All 8 batch artifacts: artifactVersion = sports_decision_artifact_v3; cognitiveProtocol present; no cognitivePhases key; interrogate = {question, probe, verify}; discern = {weigh, contrast, stress}; decide = {resolve, position, justify}; synthesize = {integrate, compose, deliver} carrying the platform-operation constants. Zero retired-shape hits (no cognitivePhases, interrogate.stress, discern.test, stress.canonical).
- Protocol fields carry real model content (e.g. perceive.detect "two right-handed starters with no clear edge"; decide.position monitor).
- interrogate.probe is deterministic: null on the 8 MLB runs because starting_pitching grounded and missingSignals was empty (no missing-primary follow-up). On the NFL smoke run (028e433e) missingSignals was {market, sharp_public} and probe populated deterministically with the market template: "Market signal missing; directional read should rely on schedule and situational context." Both the populated and null paths are confirmed correct.
- synthesize is acknowledged as platform-owned (constant "platform operation: ..." strings), not model-emitted.

### reporter (end-to-end confirmed)

20260529-1421-mlb-calibration.md rendered with the canonical "Cognitive Protocol Completeness" and "Cognitive Protocol Quality" tables over the 12 canonical micro-actions, the canonical/synthesize note, 0 retired tokens, and 0 non-ascii bytes. The Canonical Calibration Reporter v1 changes are verified against live data.

### Tool Gateway

Every run (8 batch + smoke full chain) completed, so the analysis.sports.matchup_read tool was authorized and invoked through the gateway at the platform.analyze stage end-to-end. No gateway behavior was changed this slice.

### artifact endpoint

Read cleanly: the smoke artifact was fetched directly via GET /api/agent-runs/{id}/artifact, and the harness fetched all 8 batch artifacts via the same endpoint without error.

### blockers

None. One minor operational note: the harness must be launched with `pwsh` (PS7) due to ?./?? syntax without a version guard.

### environment left running

The fresh .NET platform API (:5007) and the pre-existing agent service (:8000) were left running; stop with scripts/dev/sports/stop-sports-dev.ps1 when done. devcore-sql left up.

### next recommended slice

Either add `#requires -version 7` to run-artifact-calibration.ps1 (tiny hygiene fix matching purge-dev-agent-runs.ps1), or Protocol Node Runner v1 groundwork (a caller that passes station ids through the now station-aware Tool Gateway). A reconciliation pass (reconcile-calibration-outcomes.ps1) once these games settle is also available.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Live Canonical v3 Calibration Batch v1 completed 2026-05-29. 8 MLB runs (+1 NFL smoke), all completed; all artifacts v3 with canonical cognitiveProtocol, no retired fields; probe deterministic (populated on missing-market NFL run, null on fully-grounded MLB runs); reporter renders canonical tables, ASCII-clean. Tool Gateway and artifact endpoint confirmed end-to-end. No code change. jera-workspace-skills untouched.

## addendum: Calibration Script PowerShell 7 Guard v1 (2026-05-29)

Tiny hygiene slice. Resolves the standing note from the live-batch slice: run-artifact-calibration.ps1 uses PowerShell 7 syntax (?. and ??) but lacked a version guard, so launching it with Windows PowerShell 5.1 produced cryptic parse errors instead of a clear message.

### change

- Added `#requires -version 7` as the first line of `scripts/dev/sports/run-artifact-calibration.ps1` (matches purge-dev-agent-runs.ps1), plus a one-line comment noting the ?./?? dependency. No logic change.
- README unchanged: it already invokes the harness with `pwsh` (not `powershell`), so it does not imply 5.1 compatibility (task 6 condition not met).

### verification

- ASCII: 0 non-ascii bytes.
- PowerShell parser: 0 errors.
- Guard behavior: Windows PowerShell 5.1 now refuses with a clear "#requires statement for Windows PowerShell 7.0 ... does not match ... 5.1" error instead of parse errors; `pwsh -DryRun` runs normally (fetched games, "would create 5 run(s). no runs created.") with no billable model calls.
- No live batch run (task 8). No runtime/protocol/gateway/confidence/posture/Angular/schema change.

### risks

Negligible. A declarative `#requires` directive only; callers already use pwsh.

### next recommended slice

Protocol Node Runner v1 groundwork (a caller that passes station ids through the station-aware Tool Gateway), or reconcile-calibration-outcomes.ps1 once the 2026-05-29 games settle.

status: Calibration Script PowerShell 7 Guard v1 implemented 2026-05-29. Added #requires -version 7 to run-artifact-calibration.ps1; ASCII 0, parser 0; 5.1 refuses cleanly, pwsh dry-run runs (no billing). README unchanged (already pwsh). jera-workspace-skills untouched.

## addendum: Protocol Node Runner v1 Groundwork (2026-05-31)

Groundwork slice. No model call, no analyzer split, no artifact-output change, no production pipeline path wired. Adds the smallest useful abstraction for a future per-station executor.

### naming decisions

Accepted all five candidate names: `IProtocolNodeRunner`, `ProtocolNodeRunner`, `ProtocolNodeExecutionContext`, `ProtocolNodeExecutionResult`, `ProtocolNodeExecutionStatus`. "Execution" here means station resolution + tool-access preparation, NOT model execution (documented in the file header). Placed in DevCore.Api.Protocols next to ProtocolRegistry / ProtocolToolAccessPolicy, reusing the policy for tool authorization.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProtocolNodeRunner.cs` -- NEW. IProtocolNodeRunner + ProtocolNodeRunner + ProtocolNodeExecutionContext (StationId + reserved ToolInvocationContext) + ProtocolNodeExecutionResult (Status, Station, AllowedTools) + ProtocolNodeExecutionStatus (Resolved | UnknownStation). Static Default built from the canonical manifests.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- register IProtocolNodeRunner singleton wired to ProtocolRegistry.Default + the policy + the tool registry (no Program.cs change).
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolNodeRunnerTests.cs` -- NEW. 7 tests.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- +1 test resolving IProtocolNodeRunner through the real Program.cs graph.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### runner behavior

- Resolve(stationId | ProtocolNodeExecutionContext) -> ProtocolNodeExecutionResult. Known station -> Status=Resolved with the card and its AllowedTools. Unknown station -> Status=UnknownStation, null card, empty tools (fail closed).
- IsToolAllowed(stationId, toolId): fail closed for an unknown station or an unregistered tool; otherwise defers to ProtocolToolAccessPolicy.IsAllowed (which enforces station-card AllowedTools membership and the exact-node-or-sentinel grant). It does not invoke the gateway or run anything.

### what is intentionally NOT wired yet

- No execute method; the runner only resolves and answers access questions. No model call, no tool invocation.
- Not called from SportsRetriever, SportsAnalyzer, or SportsComposer, or any controller. ProtocolNodeExecutionContext.ToolContext is reserved for the future executor and unused in v1.
- ToolGateway behavior, telemetry, ProtocolRegistry, and the startup guard are unchanged.

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 279 passed, 0 failed (was 271; +7 runner unit tests, +1 DI-resolution test). Existing ProtocolToolAccessPolicy tests and the startup guard still green.

### risks

Low. Additive types + one DI registration; no existing behavior changed. The runner is dormant (nothing calls it). Reusing the already-tested policy keeps authorization logic single-sourced. Reversible (delete the file + the one registration + tests).

### next slice

Give the runner an actual execute path for a deterministic station first (e.g., a memory-backed interrogate.probe behind a flag), driving a station-id call through the gateway with the prepared ProtocolNodeExecutionContext -- still without splitting the analyze call. Or wire the runner read-only into a diagnostics/inspection surface.

### Claude/Codex transfer notes

- The runner is groundwork only and dormant; do not expect any runtime/pipeline behavior change. Do not wire it into SportsRetriever/SportsAnalyzer/SportsComposer without a dedicated slice.
- Tool authorization is single-sourced in ProtocolToolAccessPolicy; the runner delegates to it. Add new station/tool grants in ProtocolRegistry + ToolRegistry, not in the runner.
- ProtocolNodeExecutionContext.ToolContext exists for the future executor (run/tenant/correlation for gateway calls) and is intentionally unused now.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Protocol Node Runner v1 Groundwork implemented 2026-05-31. Added IProtocolNodeRunner/ProtocolNodeRunner + execution context/result/status types in DevCore.Api.Protocols; resolves station cards, exposes allowed tools, delegates tool authorization to ProtocolToolAccessPolicy, fails closed for unknown stations/tools. DI-registered, not wired into any pipeline. dotnet 279 (targeted 64). No model/prompt/gateway/confidence/posture/schema/Angular change. jera-workspace-skills untouched.

## addendum: Protocol Node Runner Diagnostics v1 (2026-06-04)

Read-only diagnostics slice. No model call, no tool invocation, no artifact mutation, no analyzer split, no production pipeline path wired, no HTTP endpoint. Adds the smallest read-only inspection surface over the dormant Protocol Node Runner so a developer (today: a unit test) can confirm station resolution and tool-permission behavior are understandable before the runner gets a real execute path. This is the "wire the runner read-only into a diagnostics/inspection surface" branch the groundwork addendum named.

### skills used

- superpowers: test-driven-development (diagnostics tests written first against the canonical manifests, then the service), verification-before-completion (all results below are fresh safe-runner runs). brainstorming/systematic-debugging/frontend-design not needed (locked brief, no bug, no UI).
- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault (read code + blueprint + handoff before touching the boundary), dai-token-tight, dai-agent-handoff (this addendum shape). Pack not edited.

### naming review result

Chose Option 1 (service-level only, no HTTP controller). No established read-only dev diagnostics endpoint pattern exists (the only Dev controller, DevProvisionController, is a DB-mutating provisioning endpoint), so an endpoint was not added. Names follow the DevCore.Api.Protocols precedent:
- `IProtocolStationDiagnostics` / `ProtocolStationDiagnostics` -- read-only diagnostics service, placed next to ProtocolNodeRunner / ProtocolToolAccessPolicy.
- `ProtocolStationDiagnostic` -- flat station-card snapshot record (Found, StationId, MacroProtocol, MicroAction, Purpose, AllowedModelCall, CostClass, AllowedTools, TelemetryTags, QualityGates).
- `ProtocolToolPermissionDiagnostic` -- tool-permission answer record (StationId, ToolId, Allowed, Status).
- `ProtocolToolPermissionStatus` -- enum: Allowed | UnknownStation | UnknownTool | ToolNotOnStationCard | DeniedByPolicy.
- Methods: `DescribeStation(stationId)`, `CheckTool(stationId, toolId)`. No tool ids or station ids changed.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProtocolStationDiagnostics.cs` -- NEW. The service + interface + the two DTO records + the status enum + a static `Default` built from `ProtocolNodeRunner.Default` and `ToolRegistry.Default()`.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- register `IProtocolStationDiagnostics` singleton wired to the runner + the tool registry (no Program.cs change).
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolStationDiagnosticsTests.cs` -- NEW. 7 tests.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- +1 test resolving IProtocolStationDiagnostics through the real Program.cs graph.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### diagnostics behavior

- `DescribeStation(stationId)` -> `ProtocolStationDiagnostic`. Known station: Found=true with a flat projection of the card (station id, macro protocol, micro action, purpose, model-call, cost class, allowed tools, telemetry tags, quality gates). Unknown station: a clear not-found shape (Found=false, null card fields, empty lists).
- `CheckTool(stationId, toolId)` -> `ProtocolToolPermissionDiagnostic`. `Allowed` is authoritative -- it comes straight from `ProtocolNodeRunner.IsToolAllowed` (which defers to `ProtocolToolAccessPolicy`); the diagnostics service makes no second authorization decision. `Status` is explanatory color derived from observable facts: Allowed / UnknownStation / UnknownTool / ToolNotOnStationCard / DeniedByPolicy.
- It never touches IToolGateway, calls no model, runs no tool handler, and mutates nothing.

### compatibility behavior

- Purely additive: one new file, one DI registration, plus tests. No existing type or behavior changed. Authorization stays single-sourced in ProtocolToolAccessPolicy via the runner.
- The runner, the policy, ProtocolRegistry, the startup guard, ToolGateway, and its telemetry are all unchanged.
- Dormant: nothing in SportsRetriever / SportsAnalyzer / SportsComposer or any controller calls the diagnostics service. No public HTTP surface. No FastAPI prompt, model-call count, confidence rule, posture behavior, artifact output, or schema change. No Angular, MCP, pgvector, Azure Functions, or Kubernetes change.

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 287 passed, 0 failed (was 279; +7 diagnostics unit tests, +1 DI-resolution test). The 7 diagnostics tests cover: known station card; allowed tools surfaced; allowed=true when station/card/policy agree; false for a tool not on the card (ToolNotOnStationCard); false for an unknown tool id (UnknownTool, fail closed); unknown station described cleanly (Found=false); unknown station tool-check fails closed (UnknownStation).
- No Python change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. Additive read-only service + one DI registration; no existing behavior changed and the runner stays dormant. The Allowed decision is delegated, not re-implemented, so it cannot drift from the policy; the Status enum is explanatory only. Reversible (delete the file + the one registration + tests).

### next recommended slice

Give the runner an actual execute path for a deterministic station first (e.g. a memory-backed interrogate.probe behind a flag), driving a station-id call through the gateway with the prepared ProtocolNodeExecutionContext -- still without splitting the analyze call. If a human-facing inspection surface is later wanted, add a dev-only (env.IsDevelopment + shared-secret gated, read-only GET) endpoint over IProtocolStationDiagnostics, mirroring DevProvisionController's gating but without DB access.

### Claude/Codex transfer notes

- Diagnostics is read-only and dormant; do not expect any runtime/pipeline behavior change. Do not add an HTTP endpoint or wire it into the sports pipeline without a dedicated slice.
- Tool authorization remains single-sourced in ProtocolToolAccessPolicy; diagnostics delegates the Allowed boolean to ProtocolNodeRunner.IsToolAllowed. The Status enum is explanatory only -- do not treat it as a second permission system.
- Add new station/tool grants in ProtocolRegistry + ToolRegistry, not in the diagnostics service.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Protocol Node Runner Diagnostics v1 implemented 2026-06-04. Added IProtocolStationDiagnostics/ProtocolStationDiagnostics + ProtocolStationDiagnostic / ProtocolToolPermissionDiagnostic / ProtocolToolPermissionStatus in DevCore.Api.Protocols; read-only DescribeStation + CheckTool over the dormant runner, Allowed delegated to ProtocolToolAccessPolicy, fails closed for unknown stations/tools, no HTTP endpoint. DI-registered, not wired into any pipeline. dotnet 287 (targeted 64). No model/prompt/gateway/confidence/posture/artifact/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Protocol Node Runner Deterministic Execute Path v1 (2026-06-04)

Execute-path slice. Gives the dormant runner ONE real deterministic execution path -- interrogate.probe -- and nothing else. No analyzer split, no model call, no FastAPI/prompt change, no confidence/posture change, no artifact mutation, no Tool Gateway behavior change, no schema/Angular/MCP/pgvector/Functions/Kubernetes touch. The path is test-driven only and is NOT wired into the live sports pipeline; SportsComposer still builds probe directly via CognitiveProtocolBuilder.

### skills used

- superpowers: writing-plans / planning (designed the execute seam against real code before editing), test-driven-development (execute tests written first against the canonical manifests + the real builder), verification-before-completion (all results below are fresh safe-runner runs). systematic-debugging used briefly to triage a build-time DLL lock (a running DevCore.Api.exe dev host held the output assemblies; stopped it, not a code fault).
- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault, dai-token-tight, dai-agent-handoff. Pack not edited.

### naming decisions

- `ExecuteAsync(ProtocolNodeExecutionPayload, CancellationToken)` chosen over `TryExecuteAsync`: the result object carries an explicit Status, so Try/bool-out semantics are redundant and do not compose with async. The async signature is the stable seam for a future executor whose stations drive the gateway (blueprint section 10 marks interrogate.probe as the first station slated for a governed async memory tool); v1 does zero async work (implemented via Task.FromResult, documented in the method).
- `ProtocolNodeExecutionPayload` (StationId + optional FollowUps + reserved ToolContext) and `ProtocolNodeExecutionOutput` (Status + StationId + Output) accepted as the input/output value objects.
- Extended the existing `ProtocolNodeExecutionStatus` enum with `Executed` + `UnsupportedStation` rather than adding a near-identical twin enum, sharing the single `UnknownStation` fail-closed value across the resolve and execute paths. No station ids or tool ids changed.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProtocolNodeRunner.cs` -- extended ProtocolNodeExecutionStatus (Executed, UnsupportedStation); added ProtocolNodeExecutionPayload + ProtocolNodeExecutionOutput records; added IProtocolNodeRunner.ExecuteAsync + the implementation. The probe build delegates to CognitiveProtocolBuilder.BuildProbe (internal, same assembly) so the runner and the production compose path are single-sourced.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolNodeExecuteTests.cs` -- NEW. 6 tests.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### execute behavior

`ExecuteAsync(payload)` -> `ProtocolNodeExecutionOutput`:
- unknown station id -> Status=UnknownStation, null Output (fail closed; shares the value with Resolve).
- registered station other than interrogate.probe (model-emitted stations, the synthesize trio) -> Status=UnsupportedStation, null Output.
- interrogate.probe -> Status=Executed, Output = CognitiveProtocolBuilder.BuildProbe(payload.FollowUps). Output is null when no doctrinal probe gap exists (exact existing builder behavior -- the platform never invents probe). No model call, no tool/gateway call, no external api call, no persistence.

### what is intentionally not wired yet

- No live pipeline wiring. SportsComposer.Compose still calls CognitiveProtocolBuilder.FromAnalyzerProtocolSeed directly and has no IProtocolNodeRunner dependency (asserted by a structural test). The runner remains dormant in production.
- No second station. Only interrogate.probe executes; the synthesize trio (also deterministic) is intentionally left UnsupportedStation in v1.
- ProtocolNodeExecutionPayload.ToolContext and the CancellationToken are reserved seams for the future gateway-driven executor; v1 uses neither.
- ToolGateway, ProtocolToolAccessPolicy, ProtocolRegistry, the startup guard, the diagnostics surface, and DI registrations are all unchanged.

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 293 passed, 0 failed (was 287; +6 execute tests). The 6 cover: probe returns deterministic output from follow-up data (equals the builder); probe returns null output for null and empty follow-ups; unknown station fails closed; unsupported known station (a model station and a synthesize station) returns UnsupportedStation; execute does not depend on IToolGateway (structural reflection assertion + a live probe run); production compose path (SportsComposer) is not wired to IProtocolNodeRunner (structural reflection assertion). Existing diagnostics, runner-groundwork, and ProtocolToolAccessPolicy tests stay green.
- A build-time DLL lock from a running DevCore.Api.exe dev host was cleared (process stopped) before the runs; not a code issue. No Python change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. Purely additive: new value objects + two enum members + one method, plus tests. No existing behavior changed; the runner stays dormant in production. Probe logic is delegated, not re-implemented, so the runner and SportsComposer cannot drift. Two structural tests guard against accidental future wiring (gateway into the runner; runner into the composer). Reversible (revert the runner additions + delete the test).

### next slice

Either (a) give interrogate.probe a governed, async, memory-backed augmentation behind a per-tenant flag -- the first real use of the async seam and ProtocolNodeExecutionPayload.ToolContext, driving memory.prior_runs.search through the gateway (blueprint section 10); or (b) only when a deliberate slice approves it, wire ExecuteAsync(interrogate.probe) into SportsComposer behind a flag and prove output parity with the current direct builder call before flipping. Do not split the analyze call.

### Claude/Codex transfer notes

- The execute path is deterministic, dormant, and test-only; do not wire it into SportsComposer/SportsAnalyzer/SportsRetriever without a dedicated slice that proves probe parity first.
- Probe is single-sourced in CognitiveProtocolBuilder.BuildProbe; the runner delegates to it. Add probe template/gap logic there, not in the runner.
- ExecuteAsync is async by signature only in v1 (Task.FromResult); the seam exists for the future gateway/memory-backed executor. Do not add blocking or external calls to the deterministic path.
- UnsupportedStation is deliberate for every station except interrogate.probe, including the deterministic synthesize trio.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Protocol Node Runner Deterministic Execute Path v1 implemented 2026-06-04. Added ExecuteAsync + ProtocolNodeExecutionPayload/ProtocolNodeExecutionOutput and extended ProtocolNodeExecutionStatus (Executed, UnsupportedStation) in DevCore.Api.Protocols; interrogate.probe runs deterministically via CognitiveProtocolBuilder.BuildProbe (no model/tool/gateway/external/persistence), unknown stations fail closed, all other stations UnsupportedStation. Not wired into SportsComposer (structural test guards it). dotnet 293 (targeted 64). No analyzer split, no prompt/confidence/posture/artifact/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: ProbeRequest Contract v1 (2026-06-04)

Contract slice. Gives interrogate.probe a structured, deterministic ProbeRequest handoff alongside its existing string output, so a future platform orchestrator can decide whether to run a targeted Perceive/retrieve refresh. Probe identifies gaps; it does NOT fetch, does NOT invoke Perceive, and does NOT invoke the Tool Gateway. No analyzer split, no model call, no FastAPI/prompt change, no confidence/posture change, no artifact persistence change, no Tool Gateway behavior change, no schema/Angular/MCP/pgvector/Functions/Kubernetes touch. The contract is dormant: nothing in the production pipeline reads it.

### skills used

- superpowers: writing-plans / planning (designed the contract + single-source mapping against real code first), test-driven-development (contract, mapping, and execute-exposure tests written first), verification-before-completion (all results below are fresh safe-runner runs), systematic-debugging on standby. 
- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault (read node-specs + blueprint + builder before designing), dai-token-tight, dai-agent-handoff. Pack not edited.

### naming decisions

- `ProbeRequest` is the singular handoff object (Status + Signals), matching the doctrine "a future handoff object from Interrogate to platform orchestration"; this makes "no probe needed" a natural `Status = NoProbeNeeded` with empty Signals.
- `ProbeRequestSignal` (new sub-type) carries the per-signal detail: SignalKey, Reason, SuggestedToolId, Priority, ConfidenceEffect. It is a projection of SignalFollowUpRecord, not a duplicate of it.
- Enums as proposed: `ProbeRequestStatus { Requested, NoProbeNeeded }`, `ProbePriority { Low, Normal, High }`, `ProbeConfidenceEffect { None, Dampens, Reduces }`.
- `Reason` is a string carrying the existing SignalFollowUpRecord vocabulary (primary_signal_missing / missing_confirmation / lateral_proxy) -- reuse, not a parallel enum.
- `SuggestedToolId` is nullable and null in v1: the templated signals are sport-specific tools and the probe layer has no competition context, so the orchestrator resolves the tool later.
- Placement: DevCore.Api.AgentRuns (next to SignalFollowUpRecord and CognitiveProtocolBuilder), with the mapping in CognitiveProtocolBuilder so probe semantics stay single-owner and there is no Protocols<->AgentRuns namespace cycle. The runner (already -> AgentRuns) only exposes it.

### files changed

dai:
- `platform/dotnet/DevCore.Api/AgentRuns/ProbeRequest.cs` -- NEW. ProbeRequest + ProbeRequestSignal records, ProbeRequestStatus / ProbePriority / ProbeConfidenceEffect enums, ProbeRequest.NoneNeeded.
- `platform/dotnet/DevCore.Api/AgentRuns/CognitiveProtocolBuilder.cs` -- extracted the shared gap detection into internal MissingProbeSignals(followUps) (ordered, template-known); BuildProbe now maps that set to sentences (string output byte-identical to v1); added internal BuildProbeRequest(followUps) + the per-signal derivation (BuildProbeRequestSignal, MapConfidenceEffect).
- `platform/dotnet/DevCore.Api/Protocols/ProtocolNodeRunner.cs` -- added a nullable `ProbeRequest? ProbeRequest` to ProtocolNodeExecutionOutput (additive, defaults null) and its Executed factory; ExecuteAsync(interrogate.probe) now also builds and exposes the ProbeRequest. String Output unchanged.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRequestTests.cs` -- NEW. 7 tests.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

cognitive-factory docs: not modified. The two doctrine points the slice brief asked to capture are already present in protocol-node-specs.md -- the Interrogate ordering question -> probe -> verify (node table, "Interrogate" row) and "Probe may recommend investigation; it does not retrieve" (Boundaries section, and the Probe spec facets 1/5/11). The new "future Perceive refresh is orchestrator-owned" framing is forward-looking (the orchestrator does not exist yet), so it is recorded here in the execution handoff rather than promoted into fixed doctrine prematurely.

jera-workspace-skills: untouched.

### ProbeRequest contract summary

`ProbeRequest { ProbeRequestStatus Status; IReadOnlyList<ProbeRequestSignal> Signals; bool IsRequested; static NoneNeeded }`.
`ProbeRequestSignal { string SignalKey; string Reason; string? SuggestedToolId; ProbePriority Priority; ProbeConfidenceEffect ConfidenceEffect }`.
Mapping (CognitiveProtocolBuilder.BuildProbeRequest): the gap set is exactly MissingProbeSignals (the same ordered, template-known set BuildProbe uses), so structured and string forms cannot diverge. For each signal the strongest matching follow-up rule sets Reason + Priority -- hard-missing primary -> (primary_signal_missing, High); missing confirmation -> (missing_confirmation, Normal); lateral proxy filling the parent -> (lateral_proxy, Normal). ConfidenceEffect maps from the record's ConfidencePermission (confidence_dampened/conservative -> Dampens; confidence_reduced -> Reduces; else None). No gap -> ProbeRequest.NoneNeeded.

### behavior summary

- CognitiveProtocolBuilder.BuildProbe: unchanged string output (refactored to read the shared MissingProbeSignals set; SportsComposerTests stay green, confirming byte-identical probe in the compose path).
- CognitiveProtocolBuilder.BuildProbeRequest: new deterministic structured projection. No model/tool/gateway/external/persistence.
- ProtocolNodeRunner.ExecuteAsync(interrogate.probe): returns Status=Executed with the unchanged string Output AND the new ProbeRequest. Unknown station -> UnknownStation (Output null, ProbeRequest null); any other registered station -> UnsupportedStation (both null). Resolve/IsToolAllowed unchanged.

### what is intentionally not wired yet

- No fetch, no retrieve, no Perceive call, no Tool Gateway call. ProbeRequest is data only.
- No production pipeline consumer. SportsComposer still calls CognitiveProtocolBuilder.FromAnalyzerProtocolSeed and does not read ProbeRequest; the runner stays dormant. No persisted artifact carries ProbeRequest (it is not added to AgentRunExecutionResult or the v3 artifact this slice).
- SuggestedToolId is the seam for orchestrator tool resolution; null in v1. Priority/ConfidenceEffect are descriptive only and change no confidence rule.

### tests

- safe .NET targeted: 64 passed, 0 failed (includes SportsComposerTests -> probe string output unchanged).
- safe .NET full: 300 passed, 0 failed (was 293; +7 ProbeRequest tests). Cover: represent missing sharp_public; represent no-probe-needed (null and empty follow-ups); map confidence effect + lateral-proxy parent mapping from follow-up data; execute exposes the request without changing the string output; execute with no gap exposes NoProbeNeeded + null string; unknown station unchanged (no request); unsupported station unchanged (no request). Existing probe/runner/diagnostics/policy tests stay green.
- A build-time DLL lock check ran first (no DevCore.Api.exe host was running this time). No Python change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. Additive: a new DTO file, two enum-bearing records, one shared internal helper extraction (output-preserving), one nullable field on the execute output, plus tests. BuildProbe output is byte-identical (guarded by SportsComposerTests). The structured and string forms are single-sourced through MissingProbeSignals and cannot drift. Nothing production reads ProbeRequest; reversible (revert the builder additions + the runner field + delete ProbeRequest.cs and the test).

### next recommended slice

Introduce the platform orchestrator seam that CONSUMES a ProbeRequest read-only and decides (deterministically, behind a flag, still no fetch) whether a targeted refresh is warranted -- e.g. a ProbeRefreshDecision that maps each requested signal to a candidate retrieve tool id using competition context, without invoking the gateway yet. Only after that, a separate slice may actually drive the gateway for a single targeted refresh. Do not split the analyze call; do not wire ProbeRequest into the persisted artifact until a slice explicitly justifies persistence.

### Claude/Codex transfer notes

- ProbeRequest is deterministic, dormant, and not persisted. Do not wire it into retrieval, Perceive, or the Tool Gateway without a dedicated orchestrator slice.
- Probe semantics are single-sourced in CognitiveProtocolBuilder: MissingProbeSignals is the one gap set; BuildProbe and BuildProbeRequest both read it. Add probe gap/template logic there, never in the runner or the DTO.
- SuggestedToolId is intentionally null in v1; populate it in the orchestrator slice where competition context exists, not in the probe layer.
- The string probe Output is the contract downstream code (compose, projection) still depends on; keep it byte-identical when touching BuildProbe.

### jera-workspace-skills status

Untouched (read-only this slice).

status: ProbeRequest Contract v1 implemented 2026-06-04. Added ProbeRequest/ProbeRequestSignal + ProbeRequestStatus/ProbePriority/ProbeConfidenceEffect in DevCore.Api.AgentRuns; CognitiveProtocolBuilder.BuildProbeRequest projects the same gap set as BuildProbe (string output byte-identical, single-sourced via MissingProbeSignals); ProtocolNodeRunner.ExecuteAsync(interrogate.probe) exposes the ProbeRequest on an additive nullable field. Deterministic, fetches nothing, dormant, not persisted. dotnet 300 (targeted 64). No analyzer split, no prompt/confidence/posture/artifact-persistence/gateway/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Decision v1 (2026-06-04)

Orchestrator-seam slice. Implements the "orchestrator decides" step of the doctrine chain (probe requests -> orchestrator decides -> Tool Gateway permits -> Perceive refresh receives). A deterministic read-only service consumes a ProbeRequest + a competition code and decides whether a targeted refresh is warranted, mapping each requested signal to a candidate retrieve tool id using competition context. It does NOT fetch, does NOT invoke the Tool Gateway, mutates no ProbeRequest, mutates no artifact, and persists nothing. No analyzer split, no prompt/confidence/posture change, no schema/Angular/MCP/pgvector/Functions/Kubernetes touch. Dormant: no production pipeline path calls it.

### skills used

- superpowers: writing-plans / planning (designed the contract + the signal->tool mapping against CompetitionCatalog + ToolRegistry first), test-driven-development (mapping, fail-closed, and no-gateway tests written first), verification-before-completion (fresh safe-runner runs below), systematic-debugging on standby (proactively cleared the dev-host DLL-lock check before building; none running).
- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault (reconciled the rest_fatigue/rest_schedule vocabulary before designing), dai-token-tight, dai-agent-handoff. Pack not edited.

### naming decisions

- `ProbeRefreshDecision` (singular handoff: Status + IsRefreshWarranted + Competition + Candidates) and `ProbeRefreshCandidate` (per-signal: RequestedSignalKey + nullable CandidateToolId + Reason + Priority + ConfidenceEffect).
- `ProbeRefreshDecisionStatus { RefreshWarranted, NoRefreshNeeded, NoProbeRequested }`.
- `IProbeRefreshDecisionService` / `ProbeRefreshDecisionService` with `Decide(ProbeRequest, string competitionCode)`.
- `is_refresh_warranted` is a computed `IsRefreshWarranted` (== Status == RefreshWarranted); `candidate_tool_id` is nullable and null means "no candidate / unsupported" -- this avoids a second per-candidate enum. Reason/Priority/ConfidenceEffect are carried through from the originating ProbeRequestSignal.
- Reconciliation (important): ProbeRequest signal keys are platform-canonical (rest_schedule, market, sharp_public, starting_pitching -- produced by MissingProbeSignals). The slice brief's "rest_fatigue" is the analyzer-side alias (SportsQualityChecker.SignalAliases). The service normalizes aliases to canonical before mapping (mirroring that alias map), so both vocabularies resolve. Refreshability is gated by CompetitionCatalog.ExpectedSignalNames, which correctly yields no candidate for sharp_public+mlb, starting_pitching+nba, etc.
- Placement: DevCore.Api.Protocols (the orchestration seam over probe output, next to ProtocolNodeRunner). DI-registered like the runner/diagnostics (low-risk, deterministic, reads only static manifests).

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshDecisionService.cs` -- NEW. ProbeRefreshDecisionStatus enum, ProbeRefreshCandidate + ProbeRefreshDecision records, IProbeRefreshDecisionService + ProbeRefreshDecisionService + static Default, the alias-normalization + competition-gated signal->tool mapping with registry fail-close.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- register IProbeRefreshDecisionService singleton wired to IToolRegistry (no Program.cs change).
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshDecisionTests.cs` -- NEW. 9 tests.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- +1 test resolving IProbeRefreshDecisionService through the real Program.cs graph.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### ProbeRefreshDecision summary

`ProbeRefreshDecision { ProbeRefreshDecisionStatus Status; string Competition; IReadOnlyList<ProbeRefreshCandidate> Candidates; bool IsRefreshWarranted }`.
`ProbeRefreshCandidate { string RequestedSignalKey; string? CandidateToolId; string Reason; ProbePriority Priority; ProbeConfidenceEffect ConfidenceEffect }`.
Mapping (canonical signal + competition sport family -> retrieve tool id; null when unsupported):
- sharp_public + (nfl/ncaaf/nba/ncaamb) -> market.sharp_public.split
- starting_pitching + mlb (baseball) -> pitching.mlb.probable_starters
- rest_schedule + (nba/ncaamb) (basketball) -> schedule.basketball.rest_context
- market + (nfl/ncaaf) (football) -> market.football.spread; market + (nba/ncaamb) (basketball) -> market.basketball.spread
- everything else (signal not grounded for the competition, unknown competition, unmapped pair, or a tool id not in the registry) -> null candidate.

### decision behavior

- ProbeRequest.NoneNeeded (or empty signals) -> Status=NoProbeRequested, IsRefreshWarranted=false, empty Candidates.
- Probe requested signals, at least one maps to a registered candidate tool -> Status=RefreshWarranted; each candidate carries its CandidateToolId (or null where that signal had no tool).
- Probe requested signals but none map for the competition -> Status=NoRefreshNeeded, all CandidateToolId null.
- Every fail-closed branch returns a null candidate, never a throw: unknown competition, a signal the competition does not ground, an unmapped (signal, sport family) pair, or a mapped tool id that is somehow not in ToolRegistry.

### what is intentionally not wired yet

- No Tool Gateway call, no fetch, no retrieve, no Perceive call. The service only reads static manifests (CompetitionCatalog, ToolRegistry) to resolve and fail-close candidate tool ids.
- No production pipeline consumer. Nothing calls Decide in a run; the service is dormant (DI-registered + a resolution test only). ProbeRequest and the v3 artifact are unchanged; ProbeRefreshDecision is not persisted.
- CandidateToolId is a proposal only; actually invoking a targeted retrieve through the gateway is a later slice ("Tool Gateway permits / Perceive refresh receives").

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 310 passed, 0 failed (was 300; +9 decision tests, +1 DI-resolution test). Cover: sharp_public->split for nba and nfl; starting_pitching->probable_starters for mlb; rest signal->rest_context for nba via both the canonical key and the rest_fatigue alias; market->football/basketball spread by sport; unsupported signal and wrong-competition signal both yield null candidate + NoRefreshNeeded; NoProbeNeeded -> NoProbeRequested; reason/priority/confidence_effect carried through; and a structural assertion that the service has no IToolGateway dependency. Existing probe-request/runner/diagnostics/policy tests stay green.
- No DevCore.Api.exe host was running (lock check ran first). No Python change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. Additive: one new service file + one DI registration + tests. Deterministic, reads only static manifests, no gateway/fetch/persistence. Fail-closed at every mapping branch. The structural no-gateway test guards against future accidental wiring. Reversible (delete the service + the registration + tests). Note: the alias map is a small second copy of SportsQualityChecker.SignalAliases; if the canonical signal vocabulary changes, update both (a future slice could promote a single shared signal-alias source).

### next recommended slice

The "Tool Gateway permits / Perceive refresh receives" step: a still-flagged, still-dormant executor that takes a RefreshWarranted ProbeRefreshDecision and drives ONE candidate tool through the Tool Gateway with a proper ToolInvocationContext (platform.retrieve), returning the fetched context without merging it into the artifact yet. Keep it off the production pipeline and behind a flag; prove gateway authorization + telemetry before any artifact merge. Do not split the analyze call.

### Claude/Codex transfer notes

- ProbeRefreshDecisionService is deterministic, dormant, and not persisted. It proposes tool ids; it does not permit or fetch. Do not wire it into retrieval or the gateway without the dedicated executor slice.
- Signal keys flowing from a real ProbeRequest are platform-canonical (rest_schedule, not rest_fatigue). The service normalizes analyzer aliases defensively; keep the alias map aligned with SportsQualityChecker.SignalAliases if either changes.
- Refreshability is competition-gated by CompetitionCatalog.ExpectedSignalNames; add new (signal, sport family) -> tool mappings in ResolveCandidateToolId and the corresponding ExpectedSignalNames, not in the runner or probe layer.
- The candidate tool id is confirmed against ToolRegistry (metadata read, not a gateway call) so an unmapped or stale id fails closed to null.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Probe Refresh Decision v1 implemented 2026-06-04. Added IProbeRefreshDecisionService/ProbeRefreshDecisionService + ProbeRefreshDecision/ProbeRefreshCandidate/ProbeRefreshDecisionStatus in DevCore.Api.Protocols; deterministic Decide(ProbeRequest, competitionCode) maps canonical signals (alias-normalized) to candidate retrieve tool ids gated by CompetitionCatalog.ExpectedSignalNames, fails closed to null candidate, no gateway/fetch/persistence. DI-registered, dormant. dotnet 310 (targeted 64). No analyzer split, no prompt/confidence/posture/artifact/gateway/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Tool Authorization v1 (2026-06-04)

Authorization-seam slice. Implements the "policy permits" step of the doctrine chain (probe requests -> decision selects -> policy permits -> executor fetches later -> perceive receives later). A deterministic read-only service takes a ProbeRefreshDecision candidate + a protocol node and decides whether the candidate tool is authorized, using ProtocolToolAccessPolicy as the authority. It makes NO Tool Gateway call, NO handler call, NO fetch, mutates nothing, persists nothing. No analyzer split, no prompt/confidence/posture change, no schema/Angular/MCP/pgvector/Functions/Kubernetes touch. Dormant: no production pipeline path calls it.

### pre-coding repo-state check

Verified before any change: dai and dai-vault both clean and in sync with origin; Probe Refresh Decision v1 (`7b356b3`) committed and tracked. Proceeded.

### skills used

- superpowers: writing-plans / planning (designed the authorization algorithm against the real policy + station cards before coding), test-driven-development (all status branches tested first), systematic-debugging on standby (dev-host DLL-lock check ran first; none running), verification-before-completion (fresh safe-runner runs below).
- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault -- drove the candidate-tool vs probe-card architecture reconciliation BEFORE coding (exactly its purpose: read repo + vault, reconcile, do not hack). dai-token-tight, dai-agent-handoff. Pack not edited.
- Skill sharpening note: dai-grill-with-vault earned its keep here by forcing the mismatch to the surface instead of silently patching the manifest. No weakness found this slice. A future sharpening could add an explicit "permission-matrix change = STOP and report" checklist item, since that was the load-bearing guardrail.

### architecture finding (the candidate-tool vs probe-card mismatch -- decided, not hacked)

interrogate.probe's station card has empty AllowedTools and AllowedModelCall = None (applicable stage sentinel = null). The candidate refresh tools are platform.retrieve tools (AllowedProtocolNodes = { platform.retrieve }). Decision, as principal engineer: do NOT add the refresh tools to the probe card and do NOT modify ToolRegistry. Either change would (a) contradict doctrine "Probe may recommend investigation; it does not retrieve" (protocol-node-specs.md) and (b) widen the live Tool Gateway permission matrix (adding interrogate.probe to a retrieve tool's AllowedProtocolNodes), which is explicitly out of bounds. The doctrinally correct authorizing node for a refresh is the retrieve executor stage platform.retrieve, where these tools are already permitted. So authorizing against the DEFAULT interrogate.probe correctly fails closed with ToolNotOnStationCard (probe requests, it does not retrieve); the future executor authorizes against platform.retrieve. Zero manifest changes; fail-closed preserved. Both halves are proven by tests.

### naming decisions

- `ProbeRefreshAuthorization` (per-candidate row: StationId, RequestedSignalKey, CandidateToolId, IsAuthorized, Status, Reason), `ProbeRefreshAuthorizationResult` (wrapper: StationId + Authorizations + AnyAuthorized), `ProbeRefreshAuthorizationStatus` (Authorized, NoCandidateTool, UnknownStation, UnknownTool, ToolNotOnStationCard, DeniedByPolicy, NoRefreshWarranted).
- `IProbeRefreshAuthorizationService` / `ProbeRefreshAuthorizationService` with `Authorize(ProbeRefreshDecision, string? stationId = null)` and `AuthorizeCandidate(ProbeRefreshCandidate, string stationId)`.
- The contract's station_id is a protocol node (a station id OR a documented platform sentinel), defaulting to interrogate.probe. No station ids or tool ids changed.
- Placement: DevCore.Api.Protocols, next to ProbeRefreshDecisionService and ProtocolToolAccessPolicy. DI-registered like the runner/diagnostics/decision seams.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshAuthorizationService.cs` -- NEW. ProbeRefreshAuthorizationStatus enum, ProbeRefreshAuthorization + ProbeRefreshAuthorizationResult records, IProbeRefreshAuthorizationService + ProbeRefreshAuthorizationService + static Default. Uses ProtocolToolAccessPolicy as the final authority; decomposes failure reasons via station-card + ToolRegistry + DocumentedStageSentinels.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- register IProbeRefreshAuthorizationService singleton (no Program.cs change).
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshAuthorizationTests.cs` -- NEW. 9 tests.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- +1 resolution test.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

No manifest files changed: ProtocolRegistry (station cards) and ToolRegistry are untouched -- the architecture finding above is why.

jera-workspace-skills: untouched.

### authorization contract summary

`ProbeRefreshAuthorization { string StationId; string? RequestedSignalKey; string? CandidateToolId; bool IsAuthorized; ProbeRefreshAuthorizationStatus Status; string Reason }`.
`ProbeRefreshAuthorizationResult { string StationId; IReadOnlyList<ProbeRefreshAuthorization> Authorizations; bool AnyAuthorized }`.
AuthorizeCandidate(candidate, node) algorithm: null tool -> NoCandidateTool; node not a station and not a documented sentinel -> UnknownStation; tool not registered -> UnknownTool; registered station whose card omits the tool -> ToolNotOnStationCard; else ProtocolToolAccessPolicy.IsAllowed decides Authorized vs DeniedByPolicy. Authorize(decision) returns NoRefreshWarranted when decision.Status != RefreshWarranted, otherwise one row per candidate.

### authorization behavior

- decision not RefreshWarranted -> single NoRefreshWarranted row, AnyAuthorized false.
- candidate with null tool -> NoCandidateTool.
- unknown node -> UnknownStation; unknown tool -> UnknownTool (both fail closed).
- interrogate.probe + a candidate retrieve tool -> ToolNotOnStationCard (correct: probe does not retrieve).
- platform.analyze (or any sentinel the tool does not list) + a retrieve tool -> DeniedByPolicy.
- platform.retrieve + a retrieve tool the policy permits -> Authorized. The Authorized result is always gated by ProtocolToolAccessPolicy.IsAllowed -- the service never authorizes anything the policy would deny.

### what is intentionally not wired yet

- No Tool Gateway call, no handler call, no fetch, no Perceive call. The seam decides; it does not permit-at-runtime or fetch.
- No production pipeline consumer; DI-registered + a resolution test only. ProbeRequest, ProbeRefreshDecision, and the v3 artifact are unchanged; nothing is persisted.
- No manifest change: interrogate.probe's card and the retrieve tools' AllowedProtocolNodes are untouched (see the architecture finding).

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 320 passed, 0 failed (was 310; +9 authorization tests, +1 DI-resolution test). Cover: Authorized at platform.retrieve; NoRefreshWarranted for a non-warranted decision; NoCandidateTool for a null tool; UnknownStation; UnknownTool; ToolNotOnStationCard for interrogate.probe (the architecture rule); DeniedByPolicy at platform.analyze; an end-to-end test proving default-probe fails closed while the retrieve stage authorizes; and a structural assertion that the service has no IToolGateway dependency. Existing ProbeRefreshDecision and ProtocolToolAccessPolicy tests stay green.
- No DevCore.Api.exe host was running (lock check ran first). No Python change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. Additive: one new service file + one DI registration + tests, no manifest change. ProtocolToolAccessPolicy remains the single authority; the service only decomposes failure reasons and never authorizes anything the policy denies. Fail-closed at every branch. Structural no-gateway test guards future wiring. Reversible (delete the service + the registration + tests). The deliberate non-change to the probe card means a real targeted refresh must run at platform.retrieve, not as a probe-station tool call -- recorded here so the executor slice builds the right caller context.

### next slice

The "executor fetches later" step: a flagged, still-dormant executor that takes an Authorized ProbeRefreshAuthorization (node = platform.retrieve) and drives that ONE candidate tool through the Tool Gateway with a proper ToolInvocationContext (ProtocolNode = platform.retrieve, run/tenant/correlation), returning the fetched context WITHOUT merging it into the artifact yet. Prove gateway authorization + telemetry on the targeted call before any merge; keep it behind a flag and off the production pipeline. Then a later slice handles Perceive-refresh merge -> Discern re-weigh. Do not split the analyze call.

### Claude/Codex transfer notes

- The authorization seam is deterministic, dormant, and not persisted. It decides; it does not permit-at-runtime or fetch. Do not wire it into the gateway or retrieval without the dedicated executor slice.
- The authorizing node for a refresh is platform.retrieve, NOT interrogate.probe. interrogate.probe is the requesting/provenance station and correctly fails closed (ToolNotOnStationCard). Do NOT "fix" that by adding retrieve tools to the probe card or adding interrogate.probe to the tools' AllowedProtocolNodes -- that contradicts doctrine and changes the live gateway matrix.
- ProtocolToolAccessPolicy is the single authority; the service decomposes reasons but defers the allow/deny decision to it. Add new station/tool grants in ProtocolRegistry + ToolRegistry, not here.
- The executor slice should construct a ToolInvocationContext with ProtocolNode = platform.retrieve and pass it to IToolGateway.InvokeAsync; the authorization seam already confirms that node authorizes the candidate.

### jera-workspace-skills status

Untouched (read-only this slice). Sharpening idea logged above (an explicit "permission-matrix change = STOP" item for dai-grill-with-vault); not applied without approval.

status: Probe Refresh Tool Authorization v1 implemented 2026-06-04. Added IProbeRefreshAuthorizationService/ProbeRefreshAuthorizationService + ProbeRefreshAuthorization/ProbeRefreshAuthorizationResult/ProbeRefreshAuthorizationStatus in DevCore.Api.Protocols; deterministic AuthorizeCandidate/Authorize defers the allow/deny decision to ProtocolToolAccessPolicy, fails closed for unknown node/tool, tool-not-on-card, and policy denial. Architecture decision: refresh tools authorize at platform.retrieve, NOT interrogate.probe (probe does not retrieve) -- zero manifest change, fail-closed preserved. DI-registered, dormant. dotnet 320 (targeted 64). No analyzer split, no prompt/confidence/posture/artifact/gateway/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Executor v1 (2026-06-04)

Executor slice -- the "retrieve fetches" step of the doctrine chain (probe requests -> decision selects -> authorization confirms -> retrieve fetches -> perceive receives later). A dormant orchestrator takes an Authorized ProbeRefreshAuthorization and invokes EXACTLY ONE candidate retrieve tool through the Tool Gateway with ProtocolNode = platform.retrieve, returning the fetched context as a value object. It MERGES NOTHING into the artifact, persists nothing, re-runs no cognitive station, and is disabled by default. No analyze split, no prompt/confidence/posture change, no schema/Angular/MCP/pgvector/Functions/Kubernetes touch.

### pre-coding repo-state check

Verified before any change: dai and dai-vault both clean and in sync with origin; Probe Refresh Tool Authorization v1 (`bde457a`) committed and tracked. Proceeded.

### skills used

- superpowers: writing-plans / planning (designed the per-tool typed dispatch + flag seam against the real gateway/handlers before coding), test-driven-development (all status branches tested first with a recording fake gateway), systematic-debugging on standby (dev-host DLL-lock check ran first; none running), verification-before-completion (fresh safe-runner runs below).
- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault -- used to verify the typed-input contract was constructible from existing context BEFORE committing to the design (the slice's stated stop-or-proceed risk). Confirmed: all candidate retrieve tools take a simple matchup record, so no stop-and-report was needed. dai-token-tight, dai-agent-handoff. Pack not edited.
- Skill sharpening note: dai-grill-with-vault again earned its keep by front-loading the "can the input be built cleanly?" question. No weakness found. Same sharpening idea as last slice stands (an explicit input-contract / permission-matrix STOP checklist item); not applied without approval.

### input-contract finding (the stated stop-or-proceed gate)

The candidate retrieve tools' typed handler inputs are clean and uniform: MatchupRetrievalInput(Competition, HomeTeam, AwayTeam, GameDate) for schedule.basketball.rest_context and market.sharp_public.split; MarketSpreadInput(same shape) for market.football.spread and market.basketball.spread; MlbProbableStartersInput(HomeTeam, AwayTeam, GameDate) for pitching.mlb.probable_starters. All are constructible from a matchup context, so the executor proceeds (no hack, no stop-and-report). Outputs are heterogeneous (BasketballScheduleContext?, SharpPublicLookupResult, FootballMarketContext?, BasketballMarketContext?, MlbStarterContext?), so the executor type-erases them into ProbeRefreshFetchedContext.

### naming decisions

- `IProbeRefreshExecutor` / `ProbeRefreshExecutor`, `ProbeRefreshExecutionRequest` (matchup fields + optional run/tenant/correlation), `ProbeRefreshExecutionResult`, `ProbeRefreshExecutionStatus { Executed, Disabled, NotAuthorized, NoCandidateTool, UnsupportedTool, ToolInvocationFailed }`, `ProbeRefreshFetchedContext` (ToolId, ContextType, Payload).
- Added `ProbeRefreshExecutorOptions(bool Enabled = false)` as the config/flag seam, default DISABLED. The executor is registered Scoped (not Singleton) because it depends on the scoped IToolGateway; there is therefore no static Default (tests construct it with a fake gateway). No station/tool ids changed.
- Placement: DevCore.Api.Protocols, next to the decision/authorization seams.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshExecutor.cs` -- NEW. The options record, status enum, ProbeRefreshFetchedContext / ProbeRefreshExecutionRequest / ProbeRefreshExecutionResult records, IProbeRefreshExecutor + ProbeRefreshExecutor with the per-tool typed gateway dispatch.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- register IProbeRefreshExecutor scoped + disabled (no Program.cs change).
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshExecutorTests.cs` -- NEW. 8 tests with a recording fake gateway.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- +1 resolution test.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.

No manifest or gateway/policy change: ToolGateway, ProtocolToolAccessPolicy, ToolRegistry, ProtocolRegistry are untouched. The gateway remains the runtime authority.

### executor contract summary

`ExecuteAsync(ProbeRefreshAuthorization authorization, ProbeRefreshExecutionRequest request, ct)` -> `ProbeRefreshExecutionResult { Status; RequestedSignalKey; CandidateToolId; ProtocolNode (always platform.retrieve); FetchedContext (ProbeRefreshFetchedContext?); Reason; ErrorMessage; IsExecuted }`. Order of gates: disabled -> Disabled; authorization not Authorized -> NotAuthorized; null candidate -> NoCandidateTool; unsupported tool -> UnsupportedTool; missing matchup input -> ToolInvocationFailed (no gateway call); else build typed input, call gateway at platform.retrieve, Executed (or ToolInvocationFailed if the gateway throws).

### executor behavior

- Disabled by default: with Enabled=false the executor never calls the gateway and returns Disabled. Production DI registers it disabled.
- Only an Authorized authorization with a supported candidate tool and present matchup input reaches the gateway. The gateway is always called with ProtocolNode = platform.retrieve and NEVER with interrogate.probe.
- On success: Executed with FetchedContext = { ToolId, runtime ContextType, type-erased Payload }. On a gateway throw (handler error, or the gateway's own policy re-check denying the tool -- an authorization/gateway disagreement): ToolInvocationFailed with the exception message, not a silent fetch. ToolGateway telemetry is preserved (the executor adds no logging and changes none).

### supported tools in v1

schedule.basketball.rest_context, market.sharp_public.split, market.football.spread, market.basketball.spread, pitching.mlb.probable_starters -- exactly the candidate retrieve tools ProbeRefreshDecision can emit, each with a clean typed input. Any other tool id (reference/analyze tools, unknown ids) is UnsupportedTool and never reaches the gateway.

### what is intentionally not wired yet

- No artifact merge: the fetched context is returned, never written to the artifact or persisted. The merge -> Discern re-weigh -> Decide update chain is a later slice.
- No production pipeline consumer; DI-registered (scoped, disabled) + a resolution test only. No Perceive/Discern/Decide re-run, no analyze split.
- Flag wiring to real config (appsettings) is deferred; the seam (ProbeRefreshExecutorOptions) exists and defaults disabled. Enabling it in a real environment would make a real platform.retrieve fetch -- intended future behavior, not this slice.

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 329 passed, 0 failed (was 320; +8 executor tests, +1 DI-resolution test). Cover: no gateway call when not Authorized; no gateway call when disabled; gateway called with platform.retrieve when Authorized; gateway never called with interrogate.probe; fetched context returned from a fake gateway response (type-erased with ContextType); ToolInvocationFailed when the gateway throws; UnsupportedTool for a tool with no v1 input contract; NoCandidateTool for a null candidate. Existing ProbeRefreshAuthorization and ToolGateway tests stay green.
- Full runner returned a normal pass (no "Build FAILED with 0 Error(s)" hang). No DevCore.Api.exe host was running (lock check ran first). No Python change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low-to-moderate. This is the first seam in the chain that CAN call the Tool Gateway, so the guardrails carry the weight: disabled by default, gateway only after Authorized, fixed platform.retrieve node, no merge/persistence. The gateway re-checks the policy independently, so the executor cannot fetch anything platform.retrieve does not permit (defense in depth). Risk if a future caller enables the flag in production without the merge/observability story -- mitigated by default-disabled and the dormant registration. Reversible (delete the executor + the registration + tests). The type-erased Payload (object?) is a deliberate v1 simplification; the merge slice will route by ContextType/ToolId.

### next slice

The "perceive receives" step: a still-dormant, flagged merger that takes a ProbeRefreshFetchedContext and produces an updated Perceive/retrieval view WITHOUT mutating the persisted artifact (a new derived view or a copy), proving the fetched context can re-enter the read. Then, separately, Discern re-weigh and Decide posture update. Keep confidence/posture/persistence untouched until a slice explicitly scopes them. Do not split the analyze call. Also consider wiring ProbeRefreshExecutorOptions to real config (appsettings) behind a per-tenant flag, still default disabled.

### Claude/Codex transfer notes

- The executor is dormant (disabled by default) and merges nothing. It is the first chain seam that can call the gateway -- do not enable it in production without the merge + observability slice.
- It always calls the gateway at platform.retrieve and never at interrogate.probe; the architecture boundary from the previous slice holds. Do not add a path that passes a cognitive station id to the gateway for a retrieve tool.
- Supported tools are the five candidate retrieve tools with clean typed input. Adding a tool means adding a typed dispatch arm (input record + InvokeAsync<TInput,TOutput>) and the tool to SupportsTool, nothing else.
- The gateway remains the runtime authority and re-checks the policy; an Authorized authorization that the gateway would deny surfaces as ToolInvocationFailed (the architecture-guard "report the mismatch" path), never a silent fetch.
- The executor is Scoped (depends on the scoped IToolGateway); do not make it a singleton.

### jera-workspace-skills status

Untouched (read-only this slice). Sharpening idea logged (input-contract / permission-matrix STOP checklist for dai-grill-with-vault); not applied without approval.

status: Probe Refresh Executor v1 implemented 2026-06-04. Added IProbeRefreshExecutor/ProbeRefreshExecutor + ProbeRefreshExecutionRequest/Result/Status + ProbeRefreshFetchedContext + ProbeRefreshExecutorOptions in DevCore.Api.Protocols; invokes exactly one Authorized candidate retrieve tool through ToolGateway at platform.retrieve (never interrogate.probe), returns type-erased fetched context, merges/persists nothing, disabled by default. Five supported retrieve tools; UnsupportedTool/NotAuthorized/Disabled/NoCandidateTool/ToolInvocationFailed fail closed. DI scoped + disabled, dormant. dotnet 329 (targeted 64). No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Perceive Intake v1 (2026-06-04)

Perceive-intake slice -- the "perceive receives" step of the doctrine chain (probe requests -> decision selects -> authorization confirms -> retrieve fetches -> perceive receives -> discern re-weighs later -> decide updates later). This slice adds a dormant, non-mutating intake seam that converts a `ProbeRefreshFetchedContext` or `ProbeRefreshExecutionResult` into a derived `PerceiveRefreshView`. It is not artifact merge, not persistence, not a production pipeline change, not an analyze split, and not a confidence or posture change.

### picked up from claude partial handoff

This was picked up from a partial Claude handoff after usage limits stopped the prior session halfway through the slice. The existing partial work was inspected before editing and continued because it was coherent, in scope, and already aligned with the executor contract.

### pre-change repo state

Required status checks before editing:

- `dai`: dirty with four files:
  - `M platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs`
  - `M platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs`
  - `?? platform/dotnet/DevCore.Api/Protocols/ProbeRefreshPerceiveIntake.cs`
  - `?? platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshPerceiveIntakeTests.cs`
- `dai-vault`: clean.
- `jera-workspace-skills`: clean.

### partial work classification

- In-scope partial work: all four dirty `dai` files above. They added the intake contract/service, intake tests, and low-risk DI registration + resolution coverage.
- Unrelated: none.
- Unknown: none.
- Action taken: continued Claude's partial work; did not overwrite or revert it. Tightened comments to the workspace lowercase/ascii rule and made the nullable execution-result overload explicit.

### skills and guidance used

- Local `jera-workspace-skills` read-only guidance:
  - `dai-grill-with-vault`: applied manually to check repo/vault doctrine before changing code.
  - `dai-agent-handoff`: used to shape this transfer addendum.
  - `dai-token-tight`: used for concise implementation notes.
- Requested engineering guidance applied manually: planning, test-driven coverage, systematic debugging, verification before completion, and handoff documentation.
- No `jera-workspace-skills` files were modified.

### naming decisions

- Accepted Claude's boring, doctrine-aligned names:
  - `IProbeRefreshPerceiveIntake` / `ProbeRefreshPerceiveIntake`
  - `ProbeRefreshPerceiveIntakeResult`
  - `ProbeRefreshPerceiveIntakeStatus`
  - `PerceiveRefreshView`
- Statuses: `Received`, `NoContext`, `UnsupportedContext`, `InvalidPayload`.
- Contract accepts both `ProbeRefreshExecutionResult?` and `ProbeRefreshFetchedContext?`. That follows the existing executor contract: execution metadata carries requested signal/tool ids, while fetched context carries the type-erased payload.
- Placement: `DevCore.Api.Protocols`, next to the decision, authorization, and executor seams.
- DI lifetime: singleton instance via `ProbeRefreshPerceiveIntake.Default`. The service is pure and dependency-free. It is registered only as a dormant application service; no pipeline path calls it.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshPerceiveIntake.cs` -- new intake status enum, derived view/result records, interface, and deterministic implementation.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshPerceiveIntakeTests.cs` -- new coverage for no-context, all supported fetched context types, unsupported context, invalid payload, no cognitive protocol coupling, and no tool gateway dependency.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the intake as a singleton dormant seam.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds an application-service resolution test.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### intake contract summary

`IProbeRefreshPerceiveIntake` exposes:

- `Receive(ProbeRefreshExecutionResult? execution)`
- `Receive(ProbeRefreshFetchedContext? fetched, string? requestedSignalKey = null)`

`ProbeRefreshPerceiveIntakeResult` carries:

- `Status`
- `RequestedSignalKey`
- `ToolId`
- `RawContextType`
- `View`
- `Reason`
- `ErrorMessage`
- `IsReceived`

`PerceiveRefreshView` carries:

- `SignalKey`
- `ToolId`
- `ContextType`
- `HasContext`
- `PerceivedSummary`

### intake behavior

- `NoContext`: execution did not fetch, fetched context is null, or payload is null.
- `Received`: supported tool id and expected typed payload map to a derived `PerceiveRefreshView`.
- `UnsupportedContext`: non-null payload with a tool id that has no v1 intake mapping.
- `InvalidPayload`: supported tool id with a payload that is not the expected executor output type.
- Sharp/public missing results are `Received` with `HasContext = false` because the handler returns a non-null `SharpPublicLookupResult` wrapper that carries status/reason even when no grounded split exists.
- The implementation makes no tool gateway call, no model call, no external call, no persistence call, and mutates no artifact or `CognitiveProtocol`.

### supported context coverage

The v1 intake covers exactly the executor's supported tools:

- `schedule.basketball.rest_context` -> `BasketballScheduleContext` -> signal `rest_schedule`, context type `basketball_rest_context`.
- `market.sharp_public.split` -> `SharpPublicLookupResult` -> signal `sharp_public`, context type `sharp_public_split`.
- `market.football.spread` -> `FootballMarketContext` -> signal `market`, context type `football_market_spread`.
- `market.basketball.spread` -> `BasketballMarketContext` -> signal `market`, context type `basketball_market_spread`.
- `pitching.mlb.probable_starters` -> `MlbStarterContext` -> signal `starting_pitching`, context type `mlb_probable_starters`.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `SportsCollectorOutput`.
- No production pipeline consumer.
- No Perceive, Discern, Decide, or Synthesize re-run.
- No confidence rule, posture behavior, model-call count, FastAPI prompt, database schema, Tool Gateway behavior, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.
- No typed-payload normalization beyond safe pattern matching against the executor's current output shapes.

### tests

- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshPerceiveIntakeTests /m:1 /nr:false` -- pass, 12 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshExecutorTests /m:1 /nr:false` -- pass, 8 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ToolGatewayDIRegistrationTests /m:1 /nr:false` -- pass, 9 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 342 passed.
- A transient build-artifact lock occurred only when two `dotnet test` filters were launched in parallel against the same test project (`MvcTestingAppManifest.json` in use). Sequential reruns passed. Do not run these filters in parallel against this project.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The slice is additive and dormant. The main risk is future drift between executor output types and intake pattern matches; adding a new executor tool or changing an output type must update the intake mapping and tests. The summary strings are deterministic derived text and not persisted, but a later consumer must not treat them as model judgment. DI registration is low risk because the service has no dependencies and no caller path.

### next slice

Recommended next slice: Probe Refresh Discern Re-weigh v1. Consume `PerceiveRefreshView` as a derived read-side input and produce a non-mutating `DiscernRefreshAssessment` (or similarly boring name) that re-weighs evidence quality without changing confidence, posture, persistence, or the artifact. Keep it dormant and test-only until a later slice explicitly scopes artifact merge/decision update.

### claude/codex transfer notes

- The intake is a receiver/view seam only. It does not fetch, merge, persist, or re-run cognition.
- Keep the chain ordered: executor returns fetched context; intake derives a perceive-facing view; discern and decide are later derived-view slices.
- Do not wire this into production pipeline behavior without a dedicated merge/observability slice.
- If typed payload extraction becomes unclear in a future tool, stop and define the missing input/output contract instead of string-parsing payloads.
- Do not add tool gateway/model dependencies to the intake; structural tests guard this.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Perceive Intake v1 implemented 2026-06-04. Added a dormant, deterministic `IProbeRefreshPerceiveIntake`/`ProbeRefreshPerceiveIntake` seam that maps executor fetched contexts into `PerceiveRefreshView` for all five executor-supported context types, fails closed for unsupported/mismatched payloads, and mutates/persists/calls nothing. DI singleton + resolution test. dotnet 342 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Discern Re-weigh v1 (2026-06-04)

Discern re-weigh slice -- the "discern re-weighs" step of the dormant probe-refresh chain (probe requests -> decision selects -> authorization confirms -> retrieve fetches -> perceive receives -> discern re-weighs -> decide updates later -> synthesize presents later). This slice adds a dormant, non-mutating assessment seam that consumes `PerceiveRefreshView` or `ProbeRefreshPerceiveIntakeResult` and produces a derived `ProbeRefreshDiscernAssessment`. It is not artifact merge, not persistence, not production pipeline wiring, not an analyze split, and not a confidence or posture change.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `dddfbdc feat(protocol): add probe refresh perceive intake seam`.
- `dai-vault`: clean, HEAD `a04e2ea docs(protocol): document probe refresh perceive intake`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-grill-with-vault`: read repo/vault doctrine before changing code; confirmed Discern can evaluate evidence quality but this slice must not mutate artifact/confidence/posture or call gateway/model.
- `dai-agent-handoff`: used to shape this addendum.
- `dai-token-tight`: used for compact implementation/reporting notes.

Requested superpowers-style guidance applied manually: planning, naming review, test-driven coverage, systematic debugging, verification before completion, and writing-plans discipline. No `jera-workspace-skills` files were modified.

Skill sharpening recommendation: add an explicit "probe-refresh derived seam naming gate" checklist to local DAI skills in a future approved skill-maintenance session. It should require naming the stage verb (`receive`, `reweigh`, `decide`) and explicitly rejecting artifact mutation before code. Not applied here because skill edits were not approved.

### naming decisions

- `IProbeRefreshDiscernReweigh` / `ProbeRefreshDiscernReweigh`: service/seam name. Kept `Reweigh` without a hyphen to match .NET type naming.
- `ProbeRefreshDiscernAssessment`: returned derived assessment. Chosen over `DiscernRefreshAssessment` to keep the probe-refresh family prefix with decision/authorization/executor/intake seams.
- `ProbeRefreshDiscernAssessmentStatus`: `Assessed`, `NoRefreshView`, `UnsupportedSignal`, `InvalidInput`.
- `ProbeRefreshDiscernEffect`: `SupportsRead`, `WeakensRead`, `Neutral`, `NeedsHumanReview`, `Unknown`.
- Contract accepts both `ProbeRefreshPerceiveIntakeResult?` and `PerceiveRefreshView?`. The intake-result overload preserves requested signal metadata; the direct-view overload keeps the seam testable and useful.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshDiscernReweigh.cs` -- new status/effect enums, assessment record, interface, and deterministic implementation.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshDiscernReweighTests.cs` -- new tests for no view, unsupported signal, five supported context families, ungrounded view, confidence/posture non-change, no cognitive protocol mutation, and no tool gateway dependency.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the re-weigh seam as a singleton dormant service.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds application-service resolution coverage.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### discern re-weigh contract summary

`IProbeRefreshDiscernReweigh` exposes:

- `Assess(ProbeRefreshPerceiveIntakeResult? intake)`
- `Assess(PerceiveRefreshView? view, string? requestedSignalKey = null)`

`ProbeRefreshDiscernAssessment` carries:

- `Status`
- `RequestedSignalKey`
- `ToolId`
- `SignalKey`
- `Assessment`
- `Effect`
- `Limitation`
- `SupportsExistingRead`
- `WeakensExistingRead`
- `Reason`
- `ErrorMessage`
- `IsAssessed`

### re-weigh behavior

- `NoRefreshView`: no intake result, non-received intake result, or no `PerceiveRefreshView`.
- `InvalidInput`: view exists but required fields (`SignalKey`, `ToolId`, `ContextType`) are blank.
- `UnsupportedSignal`: signal key has no v1 assessment mapping.
- `Assessed`: known refreshed signal is converted into a derived assessment.
- Ungrounded views (`HasContext = false`) are assessed with `Effect = Unknown` and no support/weakening flag.
- Basketball rest and mlb probable starters produce `Neutral` because the derived view alone is not enough to determine support or contradiction.
- Sharp/public and market spread produce `NeedsHumanReview` because they require comparison against market/original read side before support or weakening can be determined.
- `SupportsRead` and `WeakensRead` are reserved in the enum but not emitted in v1 because this seam deliberately has no original read, lean side, confidence, or posture input.

### supported signal/context coverage

The v1 re-weigh seam covers the current `PerceiveRefreshView` signal keys produced by intake:

- `rest_schedule` from `basketball_rest_context`.
- `sharp_public` from `sharp_public_split`.
- `market` from `football_market_spread`.
- `market` from `basketball_market_spread`.
- `starting_pitching` from `mlb_probable_starters`.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `DiscernProtocol`.
- No confidence or posture update.
- No `Decide` update and no `Synthesize` update.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, or persistence call.
- No FastAPI prompt, model-call count, database schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshDiscernReweighTests /m:1 /nr:false` -- pass, 11 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshPerceiveIntakeTests /m:1 /nr:false` -- pass, 12 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshExecutorTests /m:1 /nr:false` -- pass, 8 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ToolGatewayDIRegistrationTests /m:1 /nr:false` -- pass, 10 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 354 passed.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The seam is additive, pure, and dormant. Main risk: consumers may overread `NeedsHumanReview` as a decision signal. It is not; it is a derived evidence-quality assessment only. Support/weakening is intentionally not determined until a later slice supplies original read/side context. Future drift risk: if `PerceiveRefreshView.SignalKey` or context labels change, update re-weigh mappings and tests.

### next slice

Recommended next slice: Probe Refresh Decide Update v1, still derived and non-mutating. It should consume `ProbeRefreshDiscernAssessment` plus explicit existing-read context and produce a `ProbeRefreshDecisionUpdate` recommendation without changing persisted artifact, confidence, posture, or synthesize output until a later merge slice explicitly scopes those mutations.

### claude/codex transfer notes

- This seam evaluates refreshed evidence quality only. It does not decide a lean, confidence, posture, or artifact update.
- Keep support/weakening conservative. Without original read side and comparison context, use `Neutral`, `Unknown`, or `NeedsHumanReview`.
- Do not add tool gateway/model dependencies to re-weigh; structural tests guard this.
- Do not wire into production behavior without a dedicated merge/observability slice.
- If a future caller wants `SupportsRead`/`WeakensRead`, first define the input contract for original read side, market side, and comparison rules. Do not infer from summary strings.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Discern Re-weigh v1 implemented 2026-06-04. Added a dormant, deterministic `IProbeRefreshDiscernReweigh`/`ProbeRefreshDiscernReweigh` seam that maps `PerceiveRefreshView` into `ProbeRefreshDiscernAssessment` for the five intake-supported signal/context families. Effects are conservative (`Neutral`, `NeedsHumanReview`, `Unknown`) and never update confidence/posture/artifact. DI singleton + resolution test. dotnet 354 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Decide Recommendation v1 (2026-06-04)

Decide recommendation slice -- the "decide recommends" step of the dormant probe-refresh chain (probe requests -> decision selects -> authorization confirms -> retrieve fetches -> perceive receives -> discern re-weighs -> decide recommends -> synthesize presents later). This slice adds a dormant, non-mutating recommendation seam that consumes `ProbeRefreshDiscernAssessment` plus explicit existing-read context and produces a derived `ProbeRefreshDecisionRecommendation`. It is not artifact merge, not persistence, not production pipeline wiring, not an analyze split, and not a confidence or posture mutation.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `d7d2e38 feat(protocol): add probe refresh discern reweigh seam`.
- `dai-vault`: clean, HEAD `bcbd329 docs(protocol): document probe refresh discern reweigh`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-agent-handoff`: used to shape this addendum and transfer notes.
- `dai-token-tight`: applied manually for compact final reporting.
- `dai-write-skill`: read as a boundary check; no skill files were edited, and runtime code was allowed only because the slice explicitly requested DAI runtime implementation.

Requested superpowers-style guidance applied manually: planning, naming review, systematic debugging, test-driven coverage, verification before completion, and writing-plans discipline. The requested "Naming and Skills Gate" was run manually: no exact local `Naming and Skills Gate` skill exists in `jera-workspace-skills`, so the gate was interpreted as repo-state verification plus explicit naming review before coding.

### naming decisions

- `IProbeRefreshDecideRecommendation` / `ProbeRefreshDecideRecommendation`: service/seam name. Chosen to name the stage verb (`decide`) and the non-mutating output (`recommendation`), not an update.
- `ProbeRefreshDecisionRecommendation`: returned value object. Chosen over `ProbeRefreshDecideRecommendationResult` because the product is a decision recommendation, not a service result wrapper.
- `ProbeRefreshExistingReadContext`: explicit current-read input. Keeps current posture/confidence/lean context outside global state and prevents inference from persisted artifact shape.
- `ProbeRefreshDecideRecommendationStatus`: `Recommended`, `NoAssessment`, `NoChangeRecommended`, `UnsupportedAssessment`, `InvalidInput`.
- `ProbeRefreshDecideConfidenceDirection`: `Increase`, `Decrease`, `Hold`, `Unknown`.
- `ProbeRefreshDecideRecommendationEffect`: `KeepPosition`, `MoveToMonitor`, `MoveToWait`, `MoveToPass`, `MoveToAvoid`, `NeedsHumanReview`, `Unknown`.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshDecideRecommendation.cs` -- new existing-read context, recommendation status/confidence/effect enums, recommendation record, interface, and deterministic implementation.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshDecideRecommendationTests.cs` -- new coverage for no assessment, neutral, human-review, unknown, weakening, supporting, high-confidence guard, unsupported assessment, invalid current position, no cognitive protocol mutation, and no tool gateway dependency.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the decide recommendation seam as a singleton dormant service.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds application-service resolution coverage.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### decide recommendation contract summary

`IProbeRefreshDecideRecommendation` exposes:

- `Recommend(ProbeRefreshDiscernAssessment? assessment, ProbeRefreshExistingReadContext? existingRead)`

`ProbeRefreshExistingReadContext` carries:

- `CurrentPosition`
- `CurrentConfidenceBand`
- `CurrentLeanSide`
- `AllowsConfidenceIncrease`

`ProbeRefreshDecisionRecommendation` carries:

- `Status`
- `RequestedSignalKey`
- `ToolId`
- `CurrentPosition`
- `RecommendedPosition`
- `ShouldChangePosition`
- `ConfidenceDirection`
- `Reason`
- `Limitation`
- `Effect`
- `ErrorMessage`
- `IsRecommended`

### recommendation behavior

- Missing/null discern assessment returns `NoAssessment`.
- Invalid discern assessment returns `InvalidInput`.
- Non-assessed/unsupported discern status returns `UnsupportedAssessment`.
- Explicit current position is required for assessed inputs and must be one of `play`, `pass`, `monitor`, `wait`, `compare`, or `avoid`.
- `Neutral` recommends holding current position with confidence direction `Hold`.
- `Unknown` fails conservative with no position change and confidence direction `Unknown`.
- `NeedsHumanReview` never recommends an aggressive posture. It moves `play` to `monitor`, moves `compare` to `wait`, and otherwise holds current position.
- `WeakensRead` recommends safer posture or confidence direction `Decrease`: `play`/`compare` move to `monitor`, `monitor` moves to `wait`, and already conservative positions hold.
- `SupportsRead` keeps current position. It never recommends `play` in v1. It may recommend confidence direction `Increase` only when the explicit existing-read context sets `AllowsConfidenceIncrease = true` and current confidence band is not `high`.
- Already-high confidence is not raised from refreshed evidence in v1.

The implementation makes no tool gateway call, no model call, no external call, no persistence call, and mutates no artifact, `CognitiveProtocol`, confidence, or posture.

### supported assessment coverage

The v1 recommendation seam handles all current `ProbeRefreshDiscernEffect` values:

- `Neutral` -> keep current position, confidence `Hold`.
- `NeedsHumanReview` -> move aggressive/compare positions to safer review positions or hold conservative positions.
- `Unknown` -> fail closed with no change.
- `WeakensRead` -> safer posture or confidence `Decrease`.
- `SupportsRead` -> hold position; optional cautious confidence `Increase` only when explicitly allowed and not already high.

Assessment statuses covered:

- `Assessed` -> effect-specific recommendation.
- `NoRefreshView`/null -> `NoAssessment`.
- `UnsupportedSignal` -> `UnsupportedAssessment`.
- `InvalidInput` -> `InvalidInput`.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `DecideProtocol`.
- No confidence mutation and no posture mutation.
- No `CognitiveProtocol.Decide` update.
- No `Synthesize` update.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, or persistence call.
- No FastAPI prompt, model-call count, database schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshDecideRecommendationTests /m:1 /nr:false` -- pass, 11 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshDiscernReweighTests /m:1 /nr:false` -- pass, 11 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshPerceiveIntakeTests /m:1 /nr:false` -- pass, 12 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshExecutorTests /m:1 /nr:false` -- pass, 8 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ToolGatewayDIRegistrationTests /m:1 /nr:false` -- pass, 11 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 366 passed.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The seam is additive, pure, and dormant. Main risk: future callers could misread a recommendation as permission to mutate the persisted decision. This slice returns a value object only and intentionally avoids `CognitiveProtocol` inputs/outputs. Another risk is overusing `SupportsRead`; v1 only allows a cautious confidence-direction recommendation when explicit context permits it and never moves to `play`.

### next slice

Recommended next slice: Probe Refresh Synthesize Preview v1, still derived and non-mutating. Consume the decide recommendation and produce a preview-only presentation fragment that can explain "what refreshed evidence would recommend" without updating the persisted artifact, confidence, posture, or delivered result.

### claude/codex transfer notes

- This seam recommends only; it does not update `DecideProtocol`, confidence, posture, or persisted artifact state.
- Keep `ProbeRefreshExistingReadContext` explicit. Do not infer current posture/confidence/lean from global state.
- Do not wire this into production behavior without a dedicated merge/observability slice.
- Do not add tool gateway/model dependencies to decide recommendation; structural tests guard this.
- If a future slice wants real posture mutation, first define the merge authority, audit trail, and artifact-version behavior.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Decide Recommendation v1 implemented 2026-06-04. Added a dormant, deterministic `IProbeRefreshDecideRecommendation`/`ProbeRefreshDecideRecommendation` seam that maps `ProbeRefreshDiscernAssessment` plus explicit `ProbeRefreshExistingReadContext` into `ProbeRefreshDecisionRecommendation`. Behavior is conservative (`Hold`, `Decrease`, safer monitor/wait, optional cautious `Increase` only when explicitly allowed), never recommends `play`, and never updates confidence/posture/artifact. DI singleton + resolution test. dotnet 366 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Synthesize Preview v1 (2026-06-04)

Synthesize preview slice -- the "synthesize previews" step of the dormant probe-refresh chain (probe requests -> decision selects -> authorization confirms -> retrieve fetches -> perceive receives -> discern re-weighs -> decide recommends -> synthesize previews -> artifact mutation later only if explicitly scoped). This slice adds a dormant, non-mutating preview seam that consumes `ProbeRefreshPerceiveIntakeResult` or `PerceiveRefreshView`, `ProbeRefreshDiscernAssessment`, and `ProbeRefreshDecisionRecommendation`, then produces a derived `ProbeRefreshSynthesizePreviewResult`. It is not artifact merge, not persistence, not production pipeline wiring, not an analyze split, and not a confidence/posture/synthesize mutation.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `a4fd7af feat(protocol): add probe refresh decide recommendation seam`.
- `dai-vault`: clean, HEAD `eb3831b docs(protocol): document probe refresh decide recommendation`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-grill-with-vault`: used to read repo/vault doctrine before naming the preview seam.
- `dai-agent-handoff`: used to shape this addendum and transfer notes.
- `dai-token-tight`: applied manually for concise reporting.
- `dai-write-skill`: read as a boundary check; no skill files were edited, and runtime code was allowed only because the slice explicitly requested DAI runtime implementation.

Requested superpowers-style guidance applied manually: planning, naming review, systematic debugging, test-driven coverage, verification before completion, and writing-plans discipline. The requested "Naming and Skills Gate" was run manually: no exact local `Naming and Skills Gate` skill exists in `jera-workspace-skills`, so the gate was interpreted as local skill inspection, clean-state verification, and explicit naming review before coding.

Skill sharpening recommendation: add a small local DAI skill or checklist for "probe-refresh derived seam gate" in a future approved skills-maintenance session. It should require clean prior-slice state, stage-verb naming, explicit non-mutation fields, structural no-gateway/no-protocol tests, and vault handoff notes. Not applied here because `jera-workspace-skills` edits were not approved.

### naming decisions

- `IProbeRefreshSynthesizePreview` / `ProbeRefreshSynthesizePreview`: service/seam name. Chosen to name the stage verb (`synthesize`) and the preview-only output, not artifact mutation.
- `ProbeRefreshSynthesizePreviewResult`: returned value object. Chosen over `ProbeRefreshPresentationPreview` to keep the probe-refresh family prefix and station verb.
- `ProbeRefreshSynthesizePreviewStatus`: `PreviewCreated`, `NoRefreshInput`, `NoDiscernAssessment`, `NoDecisionRecommendation`, `UnsupportedInput`, `InvalidInput`.
- Result fields use preview-language names: `Headline`, `WhatChanged`, `WhyItMatters`, `RecommendationSummary`, `RemainingLimitation`, and `ShouldSurfaceToUser`.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshSynthesizePreview.cs` -- new preview status enum, result record, interface, and deterministic implementation.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshSynthesizePreviewTests.cs` -- new coverage for missing inputs, complete preview creation, intake overload, required preview fields, no cognitive/synthesize protocol mutation, no tool gateway dependency, and no changed-artifact/posture/confidence language.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the synthesize preview seam as a singleton dormant service.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds application-service resolution coverage.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### synthesize preview contract summary

`IProbeRefreshSynthesizePreview` exposes:

- `Preview(ProbeRefreshPerceiveIntakeResult? intake, ProbeRefreshDiscernAssessment? assessment, ProbeRefreshDecisionRecommendation? recommendation)`
- `Preview(PerceiveRefreshView? view, ProbeRefreshDiscernAssessment? assessment, ProbeRefreshDecisionRecommendation? recommendation, string? requestedSignalKey = null)`

`ProbeRefreshSynthesizePreviewResult` carries:

- `Status`
- `RequestedSignalKey`
- `ToolId`
- `Headline`
- `WhatChanged`
- `WhyItMatters`
- `RecommendationSummary`
- `RemainingLimitation`
- `ShouldSurfaceToUser`
- `Reason`
- `ErrorMessage`
- `IsPreviewCreated`

### synthesize preview behavior

- Missing/null/non-received perceive refresh input returns `NoRefreshInput`.
- Missing/null/no-refresh discern assessment returns `NoDiscernAssessment`.
- Missing/null/no-assessment decide recommendation returns `NoDecisionRecommendation`.
- Blank refresh view fields (`SignalKey`, `ToolId`, `ContextType`, or `PerceivedSummary`) return `InvalidInput`.
- Invalid discern or decide inputs return `InvalidInput`.
- Unsupported discern or decide status returns `UnsupportedInput`.
- Complete supported inputs return `PreviewCreated`.
- Preview text says what refreshed signal was received, what Discern assessed, what Decide recommends, and what limitation remains.
- Preview text stays conditional/preview-only. It does not say the artifact changed, posture changed, confidence changed, or that a final result was updated.
- `ShouldSurfaceToUser` is true only for grounded, non-unknown supported preview inputs. Ungrounded or unknown inputs can still produce a preview but fail conservative for surfacing.

### supported input coverage

The v1 preview supports the current derived probe-refresh outputs:

- `ProbeRefreshPerceiveIntakeResult.Status == Received` with a non-null `PerceiveRefreshView`.
- Direct `PerceiveRefreshView` for all current intake signal/context families.
- `ProbeRefreshDiscernAssessment.Status == Assessed`.
- `ProbeRefreshDecisionRecommendation` statuses that are not missing, invalid, or unsupported.

Current signal/context coverage flows through the existing intake/reweigh chain:

- `rest_schedule` / `basketball_rest_context`.
- `sharp_public` / `sharp_public_split`.
- `market` / `football_market_spread`.
- `market` / `basketball_market_spread`.
- `starting_pitching` / `mlb_probable_starters`.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `SynthesizeProtocol`.
- No confidence mutation and no posture mutation.
- No `CognitiveProtocol.Synthesize` update.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, or persistence call.
- No FastAPI prompt, model-call count, database schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshSynthesizePreviewTests /m:1 /nr:false` -- pass, 12 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshDecideRecommendationTests /m:1 /nr:false` -- pass, 11 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshDiscernReweighTests /m:1 /nr:false` -- pass, 11 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshPerceiveIntakeTests /m:1 /nr:false` -- pass, 12 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ToolGatewayDIRegistrationTests /m:1 /nr:false` -- pass, 12 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 379 passed.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The seam is additive, pure, and dormant. Main risk: future callers could present preview text as a merged artifact update. The result deliberately uses preview-only language and the seam has no `CognitiveProtocol`, `SynthesizeProtocol`, artifact, persistence, model, or gateway dependency. Future risk: `ShouldSurfaceToUser` may need product-specific policy once there is a real UI or merge path; v1 keeps that conservative.

### next slice

Recommended next slice: Probe Refresh Artifact Merge Contract v1, still non-mutating. Define the merge authority, feature flag, audit/telemetry shape, allowed mutated fields, and rollback behavior before any code updates persisted artifacts or delivered results.

### claude/codex transfer notes

- This seam previews only; it does not update `SynthesizeProtocol`, persisted artifact state, confidence, or posture.
- Keep preview text conditional. Do not state that a final result changed.
- Do not wire this into production behavior without a dedicated merge authority and audit slice.
- Do not add tool gateway/model dependencies to synthesize preview; structural tests guard this.
- If a future UI consumes the preview, keep `ShouldSurfaceToUser` conservative and product-scoped.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Synthesize Preview v1 implemented 2026-06-04. Added a dormant, deterministic `IProbeRefreshSynthesizePreview`/`ProbeRefreshSynthesizePreview` seam that maps derived perceive/discern/decide refresh outputs into `ProbeRefreshSynthesizePreviewResult`. Behavior is preview-only, conditional, and non-mutating; it never updates confidence/posture/artifact or `SynthesizeProtocol`. DI singleton + resolution test. dotnet 379 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Artifact Merge Contract v1 (2026-06-04)

Artifact merge contract slice -- the dormant governance contract after synthesize preview and before any future artifact mutation. This slice defines who/what may merge, what may change, what must never change automatically, before/after audit data, rollback representation, feature-flag gating, and tenant/run boundaries. It creates merge plans only. It does not merge, persist, mutate artifacts, update confidence/posture/lean, call a model, call the Tool Gateway, split analyze, or wire production behavior.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `fe95a15 feat(protocol): add probe refresh synthesize preview seam`.
- `dai-vault`: clean, HEAD `fda4895 docs(protocol): document probe refresh synthesize preview`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-grill-with-vault`: used to read repo/vault doctrine before naming the merge contract.
- `dai-agent-handoff`: used to shape this addendum and transfer notes.
- `dai-token-tight`: applied manually for concise reporting.
- `dai-write-skill`: read as a boundary check; no skill files were edited, and runtime code was allowed only because the slice explicitly requested DAI runtime implementation.

Requested superpowers-style guidance applied manually: planning, naming review, systematic debugging, test-driven coverage, verification before completion, and writing-plans discipline. The requested "Naming and Skills Gate" was run manually: no exact local `Naming and Skills Gate` skill exists in `jera-workspace-skills`, so the gate was interpreted as local skill inspection, clean-state verification, and explicit naming review before coding.

Skill sharpening recommendation: add a small local DAI skill or checklist for "probe-refresh artifact merge gate" in a future approved skills-maintenance session. It should require a clean prior slice, stage-verb naming, default-disabled feature flag, explicit allowed/forbidden mutation lists, before/after audit, rollback representation, structural no-gateway/no-protocol tests, and vault handoff notes. Not applied here because `jera-workspace-skills` edits were not approved.

### naming decisions

- `IProbeRefreshArtifactMergePlanner` / `ProbeRefreshArtifactMergePlanner`: service/seam name. Chosen to name plan creation, not merge execution.
- `ProbeRefreshArtifactMergePlan`: top-level returned contract. Chosen over `Contract` as the primary return type because future executors can review a concrete plan without implying mutation.
- `ProbeRefreshArtifactMergeOptions`: feature flag and authority inputs.
- `ProbeRefreshArtifactMergeContext`: explicit tenant/run/source-artifact boundary input plus optional existing field values for audit before-values.
- `ProbeRefreshArtifactMergeAudit`: before/after audit envelope.
- `ProbeRefreshMergeRollbackPlan`: rollback representation.
- `ProbeRefreshArtifactMergeStatus`: `Planned`, `Disabled`, `NotEligible`, `NoPreview`, `UnsafeChange`, `InvalidInput`.
- `ProbeRefreshMergeAuthority`: `None`, `PlatformOnly`, `TenantFlagRequired`, `ManualReviewRequired`.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshArtifactMergeContract.cs` -- new merge status/authority/change enums, options/context/audit/rollback/plan records, interface, and deterministic planner.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshArtifactMergeContractTests.cs` -- new coverage for no preview, disabled flag, planned contract, authority, allowed/forbidden categories, protected fields, audit, rollback, no context mutation, and no Tool Gateway dependency.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the planner as a singleton dormant service.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds application-service resolution coverage.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### merge contract summary

`IProbeRefreshArtifactMergePlanner` exposes:

- `Plan(ProbeRefreshSynthesizePreviewResult? preview, ProbeRefreshArtifactMergeContext? context = null, ProbeRefreshArtifactMergeOptions? options = null)`

`ProbeRefreshArtifactMergePlan` carries:

- `Status`
- `MergeAuthority`
- `FeatureFlagKey`
- `TenantKey`
- `RunId`
- `SourceArtifactVersion`
- `RequestedSignalKey`
- `AllowedChanges`
- `ForbiddenChanges`
- `ProposedChanges`
- `Audit`
- `Rollback`
- `Reason`
- `ErrorMessage`
- `IsPlanned`

Default behavior is disabled. A planned result requires a created synthesize preview, `ArtifactMergeEnabled = true`, explicit non-`None` merge authority, tenant key, run id, and source artifact version.

### allowed and forbidden changes

Allowed categories:

- `PerceiveRefreshView`
- `DiscernRefreshAssessment`
- `DecideRecommendation`
- `SynthesizePreview`
- `QualityWarning`
- `RefreshMetadata`

Forbidden categories:

- `RawRetrievedSignalsOverwrite`
- `ConfidenceMutation`
- `PostureMutation`
- `LeanMutation`
- `ArtifactVersionMutation`
- `TenantMutation`
- `RunIdMutation`
- `HistoricalAuditDeletion`

The v1 proposed field paths are restricted to `probeRefresh.*` metadata/preview paths such as requested signal key, tool id, preview headline, what changed, why it matters, recommendation summary, remaining limitation, and surfacing flag. Proposed changes deliberately do not include confidence, posture, lean, artifact version, tenant key, or run id.

### audit and rollback

`ProbeRefreshArtifactMergeAudit` records:

- tenant key
- run id
- source artifact version
- requested signal key
- proposed field path
- allowed change category
- before value
- after value

Before values come from optional `ProbeRefreshArtifactMergeContext.ExistingFieldValues`; missing before values remain null. The planner only copies values into the audit and never edits the supplied context or dictionary.

`ProbeRefreshMergeRollbackPlan` is always present. For planned changes it uses `restore_audited_before_values` and lists every proposed field path that a later merge executor would restore. This is representation only; no rollback is executed in this slice.

### feature flag and default behavior

Feature flag key:

- `ProbeRefresh:ArtifactMergeEnabled`

Default options:

- `ArtifactMergeEnabled = false`
- `MergeAuthority = ManualReviewRequired`

If no options are supplied, the planner returns `Disabled`. If the flag is enabled without valid tenant/run/source-artifact boundary data, the planner returns `InvalidInput`. If authority is `None` while enabled, the planner returns `UnsafeChange`.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `SynthesizeProtocol`.
- No confidence mutation, posture mutation, lean mutation, or artifact-version mutation.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, or persistence call.
- No FastAPI prompt, model-call count, database schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshArtifactMergeContractTests /m:1 /nr:false` -- pass, 14 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter "FullyQualifiedName~ProbeRefreshArtifactMergeContractTests|FullyQualifiedName~ProbeRefreshSynthesizePreviewTests|FullyQualifiedName~ProbeRefreshDecideRecommendationTests|FullyQualifiedName~ProbeRefreshDiscernReweighTests|FullyQualifiedName~ProbeRefreshPerceiveIntakeTests|FullyQualifiedName~ToolGatewayDIRegistrationTests" /m:1 /nr:false` -- pass, 73 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 394 passed.
- A plain multi-node `dotnet build`/`dotnet test` path returned `Build FAILED` with `0 Error(s)` before switching to the repo-documented safe runner. The safe runner exists for this exact local MSBuild behavior and passed.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The seam is additive, pure, default-disabled, and dormant. Main future risk: a merge executor could treat `Planned` as permission to mutate protected fields. The contract counters that with explicit forbidden categories, `probeRefresh.*` proposed paths only, mandatory audit and rollback representation, and tenant/run/source-artifact boundaries. Another future risk is policy ambiguity between `TenantFlagRequired` and `ManualReviewRequired`; v1 records the authority but does not enforce a real executor policy.

### next slice

Recommended next slice: Probe Refresh Merge Review/Telemetry v1, still non-mutating. Add a read-only review surface or telemetry envelope that records and inspects merge plans without applying them. Do not implement artifact mutation until review, authorization, and observability semantics are explicit.

### claude/codex transfer notes

- This seam plans only; it does not apply the plan or update persisted artifact state.
- Keep `ProbeRefreshArtifactMergeOptions` default-disabled.
- Keep proposed paths inside `probeRefresh.*` metadata unless a later slice explicitly expands the contract.
- Do not add Tool Gateway/model dependencies to the merge planner; structural tests guard this.
- Do not wire this into production behavior without a dedicated merge executor, audit persistence, and rollback test slice.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Artifact Merge Contract v1 implemented 2026-06-04. Added a dormant, deterministic `IProbeRefreshArtifactMergePlanner`/`ProbeRefreshArtifactMergePlanner` seam that maps `ProbeRefreshSynthesizePreviewResult` plus explicit context/options into `ProbeRefreshArtifactMergePlan`. Behavior is feature-flagged, audit-bearing, rollback-represented, default-disabled, and non-mutating; it never updates confidence/posture/lean/artifact or `SynthesizeProtocol`. DI singleton + resolution test. dotnet 394 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Merge Review/Telemetry v1 (2026-06-04)

Merge review/telemetry slice -- the dormant review seam after `ProbeRefreshArtifactMergePlan` and before any future artifact mutation. This slice reviews a merge plan, returns a structured telemetry event shape, and decides whether a later executor could consider automatic merge. It does not merge, persist, mutate artifacts, update confidence/posture/lean, call a model, call the Tool Gateway, split analyze, or wire production behavior.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `534b48b feat(protocol): define probe refresh artifact merge contract`.
- `dai-vault`: clean, HEAD `81b5338 docs(protocol): document probe refresh artifact merge contract`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-grill-with-vault`: used to read repo/vault doctrine before naming the review seam.
- `dai-agent-handoff`: used to shape this addendum and transfer notes.
- `dai-token-tight`: applied manually for concise reporting.
- `dai-write-skill`: read as a boundary check; no skill files were edited, and runtime code was allowed only because the slice explicitly requested DAI runtime implementation.

Requested superpowers-style guidance applied manually: planning, naming review, systematic debugging, test-driven development, verification before completion, and writing-plans discipline. The requested "Naming and Skills Gate" was run manually: no exact local `Naming and Skills Gate` skill exists in `jera-workspace-skills`, so the gate was interpreted as local skill inspection, clean-state verification, and explicit naming review before coding.

### naming decisions

- `IProbeRefreshMergeReview` / `ProbeRefreshMergeReview`: service/seam name. Chosen to name review, not execution.
- `ProbeRefreshMergeReviewResult`: returned value object carrying review status, manual-review flag, merge permission, block reasons, and telemetry event.
- `ProbeRefreshMergeReviewStatus`: `ReviewPassed`, `ReviewBlocked`, `ManualReviewRequired`, `NoMergePlan`, `Disabled`, `UnsafePlan`, `InvalidInput`.
- `ProbeRefreshMergeReviewReason`: structured block/manual-review reasons.
- `ProbeRefreshMergeTelemetryEvent`: returned telemetry shape. Chosen over structured logging because this slice should not add a metrics/logging dependency or production behavior.
- `ProbeRefreshMergeReviewContext`: correlation metadata input only.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeReview.cs` -- new review statuses, reasons, context, telemetry event, result record, interface, and deterministic reviewer.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshMergeReviewTests.cs` -- new coverage for missing plan, disabled plan, manual review, passed review, telemetry fields, forbidden confidence/posture/lean proposals, unsafe plan, no mutation, and no Tool Gateway dependency.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the review seam as a singleton dormant service.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds application-service resolution coverage.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### review behavior

`IProbeRefreshMergeReview` exposes:

- `Review(ProbeRefreshArtifactMergePlan? plan, ProbeRefreshMergeReviewContext? context = null)`

`ProbeRefreshMergeReviewResult` carries:

- `Status`
- `MayMerge`
- `RequiresManualReview`
- `TenantKey`
- `RunId`
- `CorrelationId`
- `SourceArtifactVersion`
- `RequestedSignalKey`
- `MergePlanStatus`
- `MergeAuthority`
- `BlockedReasons`
- `TelemetryEvent`
- `Reason`
- `ErrorMessage`
- `IsReviewPassed`

Default behavior is conservative: `MayMerge = false` unless the plan is `Planned`, has valid tenant/run/source-artifact/audit boundaries, proposes no forbidden field, and uses `PlatformOnly` or `TenantFlagRequired` authority. `ManualReviewRequired` authority is represented as `Status = ManualReviewRequired`, `RequiresManualReview = true`, and `MayMerge = false`.

### telemetry shape

`ProbeRefreshMergeTelemetryEvent` is returned on every review result. Event name:

- `ProbeRefreshMergeReview`

Fields:

- `RunId`
- `TenantKey`
- `CorrelationId`
- `RequestedSignalKey`
- `MergePlanStatus`
- `ReviewStatus`
- `MayMerge`
- `RequiresManualReview`
- `AllowedChangeCount`
- `ForbiddenChangeCount`
- `ProposedChangeCount`

No structured logging or metrics system was added. The event is a value object only.

### blocking rules

- Missing merge plan -> `NoMergePlan`, `MayMerge = false`.
- Disabled merge plan -> `Disabled`, `MayMerge = false`.
- Unsafe merge plan -> `UnsafePlan`, `MayMerge = false`.
- Invalid-input merge plan -> `InvalidInput`, `MayMerge = false`.
- Non-planned merge plan -> `ReviewBlocked`, `MayMerge = false`.
- Missing or mismatched tenant/run/source-artifact/audit boundary data -> `InvalidInput`, `MayMerge = false`.
- `ManualReviewRequired` authority -> `ManualReviewRequired`, `RequiresManualReview = true`, `MayMerge = false`.
- `PlatformOnly` or `TenantFlagRequired` may pass only when the plan is otherwise safe.
- Proposed `confidence`, `posture`, or `lean` field mutation always blocks automatic review.
- Proposed artifact-version, tenant, run-id, raw retrieved signals, or audit-deletion field mutation also blocks automatic review.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `SynthesizeProtocol`.
- No confidence mutation, posture mutation, lean mutation, or artifact-version mutation.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, structured log emission, metrics emission, or persistence call.
- No FastAPI prompt, model-call count, database schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshMergeReviewTests /m:1 /nr:false` -- pass, 11 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter "FullyQualifiedName~ProbeRefreshMergeReviewTests|FullyQualifiedName~ProbeRefreshArtifactMergeContractTests|FullyQualifiedName~ProbeRefreshSynthesizePreviewTests|FullyQualifiedName~ToolGatewayDIRegistrationTests" /m:1 /nr:false` -- pass, 51 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 406 passed.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The seam is additive, pure, and dormant. Main future risk: a later executor could treat `ReviewPassed` as direct permission to mutate without its own transaction/audit checks. This slice keeps `ReviewPassed` as "could consider" only and writes nothing. Another risk is field-path matching for forbidden proposals; it is conservative for the protected fields named by the contract, but a real executor should enforce typed mutation categories instead of relying only on strings.

### next slice

Recommended next slice: Probe Refresh Merge Dry-Run Executor v1, still non-mutating. Consume a merge plan plus review result and produce a would-apply diff that proves the exact artifact paths and rollback data before any persistence or artifact update is allowed.

### claude/codex transfer notes

- This seam reviews only; it does not apply a plan or emit real telemetry.
- Keep `MayMerge = false` unless status is `ReviewPassed`.
- Keep `ManualReviewRequired` separate from `ReviewPassed`; manual review is not automatic merge permission.
- Do not add Tool Gateway/model/logging/persistence dependencies to merge review; structural tests guard gateway absence.
- If a future dry-run or executor slice consumes this result, it must still re-check tenant/run/source-artifact boundaries and protected fields.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Merge Review/Telemetry v1 implemented 2026-06-04. Added a dormant, deterministic `IProbeRefreshMergeReview`/`ProbeRefreshMergeReview` seam that maps `ProbeRefreshArtifactMergePlan` plus optional correlation context into `ProbeRefreshMergeReviewResult` and `ProbeRefreshMergeTelemetryEvent`. Behavior is review-only, value-object telemetry-only, default-conservative, and non-mutating; it never updates confidence/posture/lean/artifact or emits real telemetry. DI singleton + resolution test. dotnet 406 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Merge Dry-Run Executor v1 (2026-06-05)

Merge dry-run slice -- the dormant projection seam after `ProbeRefreshMergeReview` and before any future artifact mutation. This slice consumes a reviewed `ProbeRefreshArtifactMergePlan` and projects the allowed proposed changes, audit, and rollback data that a later executor could consider. It does not merge, persist, mutate artifacts, update confidence/posture/lean, call a model, call the Tool Gateway, split analyze, or wire production behavior.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `00ccf43 feat(protocol): add probe refresh merge review seam`.
- `dai-vault`: clean, HEAD `14dc327 docs(protocol): document probe refresh merge review`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-grill-with-vault`: used to read repo/vault doctrine before naming the dry-run seam.
- `dai-agent-handoff`: used to shape this addendum and transfer notes.
- `dai-token-tight`: applied manually for concise reporting.
- `dai-write-skill`: read as a boundary check; no skill files were edited, and runtime code was allowed only because the slice explicitly requested DAI runtime implementation.

Requested superpowers-style guidance applied manually: planning, naming review, systematic debugging, test-driven development, verification before completion, and writing-plans discipline. The requested "Naming and Skills Gate" was run manually: no exact local `Naming and Skills Gate` skill exists in `jera-workspace-skills`, so the gate was interpreted as local skill inspection, clean-state verification, and explicit naming review before coding.

Skill sharpening recommendation: add a local DAI "probe-refresh merge slice gate" skill in a future approved skills-maintenance session. The repeated workflow now covers merge contract, review/telemetry, and dry-run: verify clean prior slice, choose non-mutating names, add structural no-gateway tests, run safe .NET verification, and update `current-slice.md`.

### naming decisions

- `IProbeRefreshMergeDryRunExecutor` / `ProbeRefreshMergeDryRunExecutor`: service/seam name. Chosen to name dry-run projection, not persistence or execution against storage.
- `ProbeRefreshMergeDryRunRequest`: input envelope carrying merge plan, merge review, optional source snapshot, tenant/run/correlation metadata.
- `ProbeRefreshMergeDryRunResult`: returned value object carrying status, `WouldMerge`, projected artifact/proposed changes, audit, rollback, reason, and error message.
- `ProbeRefreshMergeDryRunStatus`: `Projected`, `ReviewNotPassed`, `NoMergePlan`, `NoReview`, `UnsafePlan`, `InvalidInput`.
- `ProbeRefreshProjectedArtifact`: projection wrapper. It explicitly sets `IsFullArtifactCopy = false` because the merge plan does not yet carry enough typed artifact data to construct a safe full artifact copy.
- `ProbeRefreshProjectedChange`: projected allowed before/after field change.
- `ProbeRefreshSourceArtifactSnapshot`: optional protected-field snapshot used only to prove dry-run does not mutate or rewrite confidence/posture/lean/artifact version.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeDryRunExecutor.cs` -- new dry-run statuses, source snapshot, request, projected artifact/change records, result record, interface, and deterministic executor.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshMergeDryRunExecutorTests.cs` -- new coverage for missing plan, missing review, review not passed, `MayMerge=false`, unsafe plan, projected result, projected changes, audit, rollback, no source mutation, forbidden confidence/posture/lean proposals, and no Tool Gateway dependency.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the dry-run executor as a singleton dormant service.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds application-service resolution coverage.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### dry-run behavior

`IProbeRefreshMergeDryRunExecutor` exposes:

- `Execute(ProbeRefreshMergeDryRunRequest? request)`

`ProbeRefreshMergeDryRunResult` carries:

- `Status`
- `WouldMerge`
- `TenantKey`
- `RunId`
- `CorrelationId`
- `SourceArtifactVersion`
- `ProjectedArtifact`
- `ProjectedChanges`
- `Audit`
- `Rollback`
- `Reason`
- `ErrorMessage`
- `IsProjected`

Rules:

- Missing request -> `InvalidInput`.
- Missing merge plan -> `NoMergePlan`.
- Missing review -> `NoReview`.
- Unsafe plan/review -> `UnsafePlan`.
- Review not passed or `MayMerge=false` -> `ReviewNotPassed`.
- Non-planned merge plan with passed review -> `InvalidInput`.
- Valid planned + review-passed + `MayMerge=true` -> `Projected`, `WouldMerge=true`.

### projection behavior

The dry-run executor projects only `ProbeRefreshMergeFieldChange` values whose categories are listed in `plan.AllowedChanges` and whose field path is not protected. The current merge plan does not include enough typed artifact data to build a full copied `AgentRunExecutionResult`, so the slice intentionally returns a `ProbeRefreshProjectedArtifact` with `IsFullArtifactCopy = false` and a projected change list.

Projected changes preserve:

- allowed category
- field path
- before value
- after value
- reason

Optional `ProbeRefreshSourceArtifactSnapshot` values for artifact version, confidence, posture, and lean are carried forward unchanged on the projected artifact wrapper. They are not overwritten by projected changes.

### audit and rollback behavior

For all plan-present outcomes, the dry-run result carries through the plan's existing:

- `ProbeRefreshArtifactMergeAudit`
- `ProbeRefreshMergeRollbackPlan`

The dry-run executor does not create or persist a new audit row, and it does not execute rollback. It only surfaces the reviewable audit/rollback representation already present on the merge plan.

### protected fields

The dry-run executor does not project field paths containing:

- `confidence`
- `posture`
- `lean`
- `artifactVersion`
- `tenantKey` / `tenant`
- `runId`
- `rawRetrievedSignals`, `rawSignals`, or `retrievedSignals`
- audit deletion markers

If a review-passed input is paired with a plan that includes a protected field path, dry-run returns `UnsafePlan`, `WouldMerge=false`, and does not include the protected field in `ProjectedChanges`.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `SynthesizeProtocol`.
- No confidence mutation, posture mutation, lean mutation, or artifact-version mutation.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, structured log emission, metrics emission, persistence call, or database write.
- No FastAPI prompt, model-call count, database schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshMergeDryRunExecutorTests /m:1 /nr:false` -- pass, 14 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter "FullyQualifiedName~ProbeRefreshMergeDryRunExecutorTests|FullyQualifiedName~ProbeRefreshMergeReviewTests|FullyQualifiedName~ProbeRefreshArtifactMergeContractTests|FullyQualifiedName~ToolGatewayDIRegistrationTests" /m:1 /nr:false` -- pass, 54 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 421 passed.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The seam is additive, pure, and dormant. Main future risk: a later executor could mistake `WouldMerge=true` for write authorization. This slice names it dry-run only and constructs no full artifact copy. Another risk is protected-field matching by field path; v1 is conservative for the protected names, but a real writer should enforce typed mutation categories and transaction/audit checks.

### next slice

Recommended next slice: Probe Refresh Merge Audit Persistence Contract v1, still non-mutating. Define the durable audit row/envelope, idempotency key, tenant/run ownership checks, and rollback persistence shape before any artifact mutation is implemented.

### claude/codex transfer notes

- This seam projects only. It does not apply a plan or persist dry-run output.
- Keep `ProbeRefreshProjectedArtifact.IsFullArtifactCopy = false` until the merge plan carries typed artifact data.
- Keep `WouldMerge=false` unless status is `Projected`.
- Keep confidence/posture/lean/artifact version as protected carried-forward snapshot values, not projected mutations.
- If a future writer consumes dry-run output, it must still re-check review status, tenant/run/source-artifact boundaries, protected fields, idempotency, transaction scope, and rollback persistence.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Merge Dry-Run Executor v1 implemented 2026-06-05. Added a dormant, deterministic `IProbeRefreshMergeDryRunExecutor`/`ProbeRefreshMergeDryRunExecutor` seam that maps a reviewed merge plan into projected allowed changes plus existing audit/rollback representation. Behavior is dry-run-only, projection-only, default-blocking, and non-mutating; it never updates confidence/posture/lean/artifact or persists dry-run output. DI singleton + resolution test. dotnet 421 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Merge Audit Persistence Contract v1 (2026-06-05)

Merge audit persistence contract slice -- the dormant audit envelope after `ProbeRefreshMergeDryRunExecutor` and before any persistence writer or artifact mutation. This slice defines the record shape a future writer would persist: audit id, idempotency key, tenant/run/correlation boundaries, before/after payload references, rollback record, retention shape, protected-field blocking, and readiness status. It does not merge, persist, mutate artifacts, update confidence/posture/lean, call a model, call the Tool Gateway, split analyze, or wire production behavior.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `6e850fd feat(protocol): add probe refresh merge dry-run executor`.
- `dai-vault`: clean, HEAD `6bd8a72 docs(protocol): document probe refresh merge dry-run`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-grill-with-vault`: used to read repo/vault doctrine before naming the audit contract.
- `dai-agent-handoff`: used to shape this addendum and transfer notes.
- `dai-token-tight`: applied manually for concise reporting.
- `dai-write-skill`: read as a boundary check; no skill files were edited, and runtime code was allowed only because the slice explicitly requested DAI runtime implementation.

Requested superpowers-style guidance applied manually: planning, naming review, systematic debugging, test-driven development, verification before completion, and writing-plans discipline. The requested "Naming and Skills Gate" was run manually: no exact local `Naming and Skills Gate` skill exists in `jera-workspace-skills`, so the gate was interpreted as local skill inspection, clean-state verification, and explicit naming review before coding.

### naming decisions

- `ProbeRefreshMergeAuditRecord`: top-level durable-audit envelope. Chosen over writer/repository names because this slice defines contract only.
- `IProbeRefreshMergeAuditFactory` / `ProbeRefreshMergeAuditFactory`: pure factory for the envelope. It returns a value object only and has no persistence dependency.
- `ProbeRefreshMergeIdempotencyKey`: deterministic duplicate-write prevention key.
- `ProbeRefreshMergePayloadReference`: before/after payload reference shape.
- `ProbeRefreshMergeRollbackRecord`: rollback strategy and prior payload reference.
- `ProbeRefreshMergeAuditRetention`: explicit retention/cleanup shape.
- `ProbeRefreshMergeAuditStatus`: `Planned`, `Reviewed`, `DryRunProjected`, `ReadyForManualReview`, `ReadyForPersistCandidate`, `Blocked`, `Invalid`.

### files changed

dai:

- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeAuditPersistenceContract.cs` -- new audit statuses, request, audit record, idempotency key, payload reference, rollback record, retention record, interface, and deterministic factory.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshMergeAuditPersistenceContractTests.cs` -- new coverage for ready audit creation, tenant/run/correlation fields, deterministic idempotency, fail-closed missing boundaries, protected confidence/posture/lean blocks, before/after payload refs, rollback, retention, no input mutation, and no persistence/Tool Gateway dependency.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers the audit factory as a singleton dormant service.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- adds application-service resolution coverage.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### audit contract behavior

`IProbeRefreshMergeAuditFactory` exposes:

- `Create(ProbeRefreshMergeAuditRequest? request)`

`ProbeRefreshMergeAuditRecord` carries:

- `AuditId`
- `IdempotencyKey`
- `TenantKey`
- `RunId`
- `CorrelationId`
- `SourceArtifactVersion`
- `SourceArtifactId`
- `SourceRunId`
- `RequestedSignalKey`
- `MergePlanStatus`
- `MergeReviewStatus`
- `DryRunStatus`
- `MergeAuthority`
- `CreatedAtUtc`
- `CreatedBy`
- `AllowedChanges`
- `ForbiddenChanges`
- `ProposedChanges`
- `BeforePayload`
- `AfterPayload`
- `Rollback`
- `Retention`
- `Reason`
- `Status`
- `ErrorMessage`

Valid planned + review-passed + projected dry-run inputs produce `Status = ReadyForPersistCandidate`. This means "a later writer may consider persisting this audit envelope," not permission to mutate an artifact.

### idempotency behavior

`ProbeRefreshMergeIdempotencyKey` includes:

- tenant key
- run id
- requested signal key
- candidate tool id
- source artifact version
- proposed-change hash
- deterministic value string

The key is deterministic for the same merge input and does not include `CreatedAtUtc`, so retries do not create a different duplicate-write key. A different requested signal key creates a different idempotency key.

### tenant, run, and correlation boundaries

The factory fails closed with `Status = Invalid` when tenant key, run id, source artifact version, audit data, or matching plan/review/dry-run boundaries are missing or mismatched. Correlation id is carried as metadata from the explicit request first, then dry-run, then review; it is not used to authorize writes.

### payload references

The contract represents payloads by reference shape only:

- before payload: inline summary + deterministic hash of audited before values.
- after payload: inline summary + deterministic hash of projected after values.
- storage kind is `InlineSummary` today.
- `ExternalUri` is nullable and remains null in this contract-only slice.

No full artifact copy is constructed.

### rollback and retention

Rollback is represented as:

- rollback strategy, defaulting to `restore_audited_before_values`.
- previous payload reference pointing at the before payload.
- rollback reason.
- `RequiresManualReview = true` by default.

Retention is represented as:

- retention policy.
- optional expiration timestamp.
- `DeleteAfterMergeSuperseded` flag.

The slice only defines these values. It does not delete, expire, archive, or restore anything.

### protected fields

The audit factory blocks readiness when proposed changes target protected paths containing:

- `confidence`
- `posture`
- `lean`
- `artifactVersion`
- `tenantKey` / `tenant`
- `runId`
- `rawRetrievedSignals`, `rawSignals`, or `retrievedSignals`
- audit deletion markers

Protected-field proposals return `Status = Blocked` and `IsReadyForPersistCandidate = false`.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `SynthesizeProtocol`.
- No confidence mutation, posture mutation, lean mutation, or artifact-version mutation.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, structured log emission, metrics emission, persistence call, database write, migration, repository, or transaction.
- No FastAPI prompt, model-call count, database schema, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshMergeAuditPersistenceContractTests /m:1 /nr:false` -- pass, 19 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter "FullyQualifiedName~ProbeRefreshMergeAuditPersistenceContractTests|FullyQualifiedName~ProbeRefreshMergeDryRunExecutorTests|FullyQualifiedName~ProbeRefreshMergeReviewTests|FullyQualifiedName~ProbeRefreshArtifactMergeContractTests|FullyQualifiedName~ToolGatewayDIRegistrationTests" /m:1 /nr:false` -- pass, 74 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 441 passed.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low. The seam is additive, pure, and dormant. Main future risk: a later writer could interpret `ReadyForPersistCandidate` as permission to mutate the artifact. This slice keeps the status scoped to audit-envelope persistence consideration only and writes nothing. Another risk is protected-field matching by path string; it is conservative for the protected names, but a real writer should enforce typed mutation categories inside the transaction.

### next slice

Recommended next slice: Probe Refresh Merge Audit Store v1. Add the database entity/migration/repository for idempotent audit-record persistence only, using this contract as the input. Keep artifact mutation deferred until the audit store, idempotent upsert behavior, tenant/run transaction boundary, and rollback-read path are tested.

### claude/codex transfer notes

- This seam creates audit envelopes only. It does not persist them.
- Keep the idempotency key independent of `CreatedAtUtc`.
- Keep `ReadyForPersistCandidate` narrower than merge permission.
- Keep rollback manual-review default true.
- If a future audit store consumes this record, it must still enforce tenant/run/source-artifact boundaries, idempotency, unique constraints, transaction scope, retention policy, rollback-read support, and protected-field blocking.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Merge Audit Persistence Contract v1 implemented 2026-06-05. Added a dormant, deterministic `IProbeRefreshMergeAuditFactory`/`ProbeRefreshMergeAuditFactory` seam that maps a reviewed dry-run projection into a durable-audit envelope contract with idempotency, payload references, rollback, retention, and boundary checks. Behavior is contract-only, default-blocking, and non-mutating; it never updates confidence/posture/lean/artifact and never persists the audit record. DI singleton + resolution test. dotnet 441 full, targeted 64. No analyzer split, no prompt/confidence/posture/artifact/gateway-behavior/schema/Angular/MCP change. jera-workspace-skills untouched.

## addendum: Probe Refresh Merge Audit Store v1 (2026-06-05)

First persistence slice for the probe-refresh merge chain. This slice persists `ProbeRefreshMergeAuditRecord` rows only, idempotently by key, so future merge attempts have an audit ledger before artifact mutation exists. It does not merge, mutate artifacts, update confidence/posture/lean, call a model, call the Tool Gateway, split analyze, or wire production behavior.

### naming and skills gate

Required baseline before coding:

- `dai`: clean, HEAD `1c217569357af557ca2b0a96a8bfa078f1ca63d8`.
- `dai-vault`: clean, HEAD `8155f57e04b3cae99560e78160fbb821d2fd20bc`.
- `jera-workspace-skills`: clean.

Local skills inspected and applied manually:

- `dai-grill-with-vault`: used to read repo/vault doctrine and persistence conventions before naming the store.
- `dai-agent-handoff`: used to shape this addendum and transfer notes.
- `dai-token-tight`: applied manually for concise reporting.
- `dai-write-skill`: read as a boundary check; no skill files were edited, and runtime code was allowed only because the slice explicitly requested DAI runtime implementation.

Requested superpowers-style guidance applied manually: planning, naming review, systematic debugging, test-driven development, verification before completion, and writing-plans discipline. The requested "Naming and Skills Gate" was run manually: no exact local `Naming and Skills Gate` skill exists in `jera-workspace-skills`, so the gate was interpreted as local skill inspection, clean-state verification, persistence-pattern inspection, and explicit naming review before coding.

Skill sharpening recommendation: create a local DAI "probe-refresh persistence slice gate" skill in a future approved skills-maintenance session. The repeated workflow is now specific enough: verify prior slice commits, inspect EF/migration pattern, name dormant persistence surfaces, add idempotency/tenant-run tests, generate isolated migration, run safe .NET verification, and update `current-slice.md`.

### persistence pattern found

Persistence is EF Core:

- domain entities live in `dai/platform/dotnet/DevCore.Domain`.
- `AppDbContext` and migrations live in `dai/platform/dotnet/DevCore.Data`.
- API-side services inject `AppDbContext` directly or through scoped services.
- migrations are established under `DevCore.Data/Migrations` and model shape is configured in `AppDbContext.OnModelCreating`.

### naming decisions

- `ProbeRefreshMergeAuditEntity`: EF/domain row for the audit ledger.
- `IProbeRefreshMergeAuditStore` / `ProbeRefreshMergeAuditStore`: scoped API-side store/repository. Chosen over writer/executor names because it stores audit rows only.
- `ProbeRefreshMergeAuditStoreResult`: returned result carrying status, record, and error message.
- `ProbeRefreshMergeAuditStoreStatus`: `Inserted`, `Existing`, `Invalid`.

### files changed

dai:

- `platform/dotnet/DevCore.Domain/Agentic/ProbeRefreshMergeAuditEntity.cs` -- new EF entity for audit ledger rows.
- `platform/dotnet/DevCore.Data/AppDbContext.cs` -- `DbSet` plus table mapping, required fields, JSON columns, and indexes.
- `platform/dotnet/DevCore.Data/Migrations/20260605123908_AddProbeRefreshMergeAuditStore.cs` -- creates `ProbeRefreshMergeAudits`.
- `platform/dotnet/DevCore.Data/Migrations/20260605123908_AddProbeRefreshMergeAuditStore.Designer.cs` -- generated migration model.
- `platform/dotnet/DevCore.Data/Migrations/AppDbContextModelSnapshot.cs` -- generated snapshot update.
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeAuditStore.cs` -- new scoped store, idempotent insert logic, and entity/contract mapper.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshMergeAuditStoreTests.cs` -- focused persistence tests.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- scoped dormant store DI registration.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- store resolution test.

dai-vault:

- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills:

- untouched.

### table/entity behavior

`ProbeRefreshMergeAudits` stores:

- surrogate key: `ProbeRefreshMergeAuditKey`.
- required logical ids: `AuditId`, `IdempotencyKey`, `TenantKey`, `RunId`.
- metadata: `CorrelationId`, `SourceArtifactVersion`, `SourceArtifactId`, `RequestedSignalKey`, `CandidateToolId`, `CreatedAtUtc`, `CreatedBy`.
- status fields: `MergePlanStatus`, `MergeReviewStatus`, `DryRunStatus`, `AuditStatus`, `MergeAuthority`.
- JSON payloads: `AllowedChangesJson`, `ForbiddenChangesJson`, `ProposedChangesJson`, `BeforePayloadJson`, `AfterPayloadJson`, `RollbackJson`, `RetentionJson`.
- review text: `Reason`, nullable `ErrorMessage`.
- `ProposedChangeHash` so the idempotency record can round-trip.

The table intentionally has no foreign key to `AgentRuns`, no cascade path, and no artifact column. It is an audit ledger, not an artifact writer.

### idempotent store behavior

`IProbeRefreshMergeAuditStore` exposes:

- `StoreAsync(ProbeRefreshMergeAuditRecord? record, CancellationToken ct)`
- `GetByIdempotencyKeyAsync(string idempotencyKey, CancellationToken ct)`
- `GetByRunAsync(long tenantKey, Guid runId, CancellationToken ct)`

Store behavior:

- missing tenant key, run id, audit id, or idempotency key -> `Invalid`, no insert.
- new idempotency key -> insert row and return `Inserted`.
- duplicate idempotency key -> return existing row and `Existing`.
- duplicate writes do not overwrite the existing audit row.
- `DbUpdateException` from a race on the unique key is handled by clearing the tracker and reading the existing row.

### tenant/run boundary behavior

Tenant/run are required at the store boundary and indexed together. They are treated as the economic/artifact boundary for audit lookup. The store does not infer tenant/run from `AgentRun`, and it does not update or join artifact rows.

### indexes, constraints, and migration

Migration: `20260605123908_AddProbeRefreshMergeAuditStore`.

Generated schema changes are isolated to:

- create table `ProbeRefreshMergeAudits`.
- unique index on `AuditId`.
- unique index on `IdempotencyKey`.
- non-unique index on `{ TenantKey, RunId }`.

No unrelated schema change was generated. EF tooling note: plain `dotnet build platform/dotnet/DevCore.Api/DevCore.Api.csproj --no-restore -v minimal` returned `Build FAILED` with `0 Error(s)`. Retrying with `/m:1 /nr:false` succeeded, then `dotnet ef migrations add ... --no-build` generated the migration.

### what is intentionally not wired yet

- No artifact merge and no mutation of `SportsRunArtifact`, `AgentRun.OutputJson`, `AgentRunExecutionResult`, `CognitiveProtocol`, or `SynthesizeProtocol`.
- No confidence mutation, posture mutation, lean mutation, or artifact-version mutation.
- No production pipeline consumer.
- No Tool Gateway call, model call, external call, structured log emission, metrics emission, artifact write, or merge result write.
- No FastAPI prompt, model-call count, Angular, MCP, pgvector, Azure Functions, Kubernetes, or production secret change.

### tests

- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~ProbeRefreshMergeAuditStoreTests /m:1 /nr:false` -- pass, 15 passed.
- `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter "FullyQualifiedName~ProbeRefreshMergeAuditStoreTests|FullyQualifiedName~ProbeRefreshMergeAuditPersistenceContractTests|FullyQualifiedName~ProbeRefreshMergeDryRunExecutorTests|FullyQualifiedName~ProbeRefreshMergeReviewTests|FullyQualifiedName~ProbeRefreshArtifactMergeContractTests|FullyQualifiedName~ToolGatewayDIRegistrationTests" /m:1 /nr:false` -- pass, 90 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass, 64 passed.
- `scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass, 457 passed.
- Migration compile check: `dotnet build platform/dotnet/DevCore.Api/DevCore.Api.csproj --no-restore -v minimal /m:1 /nr:false` -- pass.
- No PowerShell files changed, so PowerShell parser/ascii validation was not required.
- No Python/FastAPI change -> pytest not run. No Angular change -> Angular build not run.

### risks

Low to medium. This is the first persistence slice, so the risk is schema inertia: table/field names will become durable. Mitigation: names explicitly say audit store, not merge executor, and the migration contains no artifact FK/write path. Another risk is JSON payload queryability; v1 chooses inspectable serialized payloads to preserve the full contract without over-modeling, and indexed lookup stays on idempotency and tenant/run.

### next slice

Recommended next slice: Probe Refresh Merge Audit Read Surface v1. Add a dev/internal read-only inspection endpoint or service for audit rows by tenant/run/idempotency so reviewers can inspect persisted audit evidence before any artifact mutation slice exists.

### claude/codex transfer notes

- The audit store is dormant but registered; no pipeline calls it.
- `IdempotencyKey` is the logical uniqueness boundary.
- `TenantKey + RunId` is the lookup boundary.
- Do not add artifact writes to this store.
- Do not overwrite rows on duplicate idempotency keys.
- If the next slice adds a read surface, keep it internal/dev-only and tenant-scoped. Do not expose merge execution.

### jera-workspace-skills status

Clean before changes and untouched after changes. Skill files were read only; no edits made.

status: Probe Refresh Merge Audit Store v1 implemented 2026-06-05. Added `ProbeRefreshMergeAudits` EF table/entity/migration and a dormant scoped `IProbeRefreshMergeAuditStore` that inserts audit records idempotently, returns existing rows for duplicate idempotency keys, preserves rollback/retention/payload JSON, and never mutates `AgentRun` artifacts. DI scoped registration + resolution test. dotnet 457 full, targeted 64. No artifact merge, no confidence/posture/lean mutation, no Tool Gateway/model/FastAPI/Angular/MCP/pgvector/Azure/Kubernetes/secrets change. jera-workspace-skills untouched.

## addendum: Local Path Placeholder Hygiene v1 (2026-06-05)

Repo-hygiene slice. No runtime code, product behavior, prompts, protocol runtime, Tool Gateway, database, Angular, artifact persistence, MCP, pgvector, Azure Functions, Kubernetes, secrets, or skill-pack files changed.

### convention

Future prompts, handoff notes, and final reports should prefer placeholders over exact local paths:

- `<DAI_WORKSPACE_ROOT>`
- `<DAI_REPO_ROOT>`
- `<DAI_VAULT_ROOT>`
- `<JERA_WORKSPACE_ROOT>`
- `<JERA_SKILLS_ROOT>`

Local agents may resolve these placeholders from a machine-specific path map such as `.local/agent-paths.md`. The `.local/` directory is gitignored and must not contain committed docs. Committed examples must use placeholders and fake paths only.

### files changed

dai:

- `.gitignore` -- ignores `.local/` for local-only agent path maps.
- `docs/examples/agent-paths.example.md` -- committed example path-map template with placeholders and fake paths only.

dai-vault:

- `.gitignore` -- ignores `.local/` for local-only agent path maps.
- `06 Execution/handoffs/current-slice.md` -- this convention note.

jera-workspace-skills:

- untouched.

### skills and validation notes

Local skills inspected and applied manually: `dai-grill-with-vault`, `dai-agent-handoff`, `dai-token-tight`, and `dai-write-skill`. The skill weakness remains the same: this repeated repo-hygiene gate would benefit from a dedicated local DAI repo-hygiene/privacy skill, but no skill files were edited.

Historical handoff entries were not rewritten. Current-slice history had pre-existing exact local path mentions; this slice adds no new exact local path or username in added lines.

Validation:

- `.local/agent-paths.md` is ignored in both `dai` and `dai-vault`.
- added lines contain no real local username or exact repo path.
- no .NET, PowerShell, Python, or Angular tests were needed because no runtime or script files changed.

### next slice

Recommended next slice remains Probe Refresh Merge Audit Read Surface v1, internal/dev-only read by idempotency key and tenant/run.

status: Local Path Placeholder Hygiene v1 implemented 2026-06-05. Added gitignored local path-map convention and a committed placeholder-only template. No real local path map committed. jera-workspace-skills untouched.

## addendum: Probe Refresh Merge Audit Read Surface v1 (2026-06-05)

Read-side slice for the probe-refresh merge audit ledger. Adds a tenant-safe, validating read facade over the committed ProbeRefreshMergeAuditStore: read one record by tenant-scoped idempotency key, or read a tenant + run history. Read-only and deterministic: no write, no merge, no artifact mutation, no AgentRun mutation, no Tool Gateway call, no model/external call, no schema change. Dormant: no production pipeline path and no HTTP endpoint added.

### pre-coding repo-state check

Verified before any change: <DAI_REPO_ROOT>, <DAI_VAULT_ROOT>, and <JERA_SKILLS_ROOT> all clean and in sync with origin. Probe Refresh Merge Audit Store v1 committed (dai 3ce4a69, "persist probe refresh merge audit records") and its files tracked. Proceeded.

### skills/guidance used

- superpowers: writing-plans / planning, test-driven-development (DB-backed tests for every status and boundary), systematic-debugging on standby (dev-host DLL-lock check ran first; none running), verification-before-completion (fresh safe-runner runs below).
- Local jera-workspace-skills/dai (read-only, guidance applied manually): dai-grill-with-vault -- inspected the committed store/contract/entity first, which surfaced that the store's GetByIdempotencyKeyAsync is NOT tenant-scoped (the load-bearing finding for this slice). dai-token-tight, dai-agent-handoff. Pack not edited.
- Skill sharpening note: dai-grill-with-vault earned its keep (read-before-build caught the non-tenant-scoped query). Standing sharpening idea: an explicit tenant-scope / cross-tenant-leak checklist item for read surfaces.

### audit store/persistence pattern found

ProbeRefreshMergeAuditStore (DevCore.Api.Protocols, scoped, AppDbContext-backed) already exposes the reads this slice needs: GetByIdempotencyKeyAsync(string idempotencyKey) returning a single record (keyed on the globally-unique idempotency key, NOT tenant-scoped) and GetByRunAsync(long tenantKey, Guid runId) returning a list tenant-scoped and ordered by CreatedAtUtc ascending then ledger key. The idempotency key embeds TenantKey, so cross-tenant key collisions are impossible. The read service reuses both methods rather than duplicating EF queries.

### naming decisions

- IProbeRefreshMergeAuditReadService / ProbeRefreshMergeAuditReadService; ProbeRefreshMergeAuditReadResult (single idempotency read); ProbeRefreshMergeAuditRunHistory (tenant+run list with Count); ProbeRefreshMergeAuditReadStatus.
- TenantMismatch deliberately REJECTED from the status enum: every path that would return it leaks cross-tenant existence, which the brief says to avoid. A cross-tenant idempotency hit returns NotFound. Statuses = Found, NotFound, InvalidInput.
- Strongly-typed long tenantKey / Guid runId (not the candidate string signatures) to match the entity and the existing store.
- ProbeRefreshMergeAuditSummary deferred: not needed for the narrow surface; RunHistory carries Count.

### files changed

dai:
- platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeAuditReadService.cs -- NEW. Read status enum, ProbeRefreshMergeAuditReadResult + ProbeRefreshMergeAuditRunHistory records, interface + service (delegates to the store; adds validation + tenant guard + status results).
- platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs -- register IProbeRefreshMergeAuditReadService scoped.
- platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshMergeAuditReadServiceTests.cs -- NEW. 14 DB-backed tests (in-memory EF).
- platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs -- +1 resolution test.

No EF schema/migration change (reuses the existing ProbeRefreshMergeAudits DbSet and store queries; no new entity, index, or model-snapshot churn). No PowerShell file change.

dai-vault: 06 Execution/handoffs/current-slice.md -- this addendum. jera-workspace-skills: untouched.

### read surface summary

- GetByIdempotencyKeyAsync(long tenantKey, string? idempotencyKey) -> ProbeRefreshMergeAuditReadResult { Status, TenantKey, IdempotencyKey, Record, Reason, ErrorMessage; IsFound }.
- GetByRunAsync(long tenantKey, Guid runId) -> ProbeRefreshMergeAuditRunHistory { Status, TenantKey, RunId, Records, Count, Reason, ErrorMessage; IsFound }.

### tenant/idempotency/run boundary behavior

- tenantKey <= 0 -> InvalidInput (both reads). Empty idempotency key -> InvalidInput. Guid.Empty run id -> InvalidInput.
- Idempotency read is tenant-scoped via a post-fetch tenant guard: the store fetches by the globally-unique key, then the service returns the record ONLY if record.TenantKey == requested tenant; otherwise NotFound. A record owned by another tenant reads as NotFound -- no cross-tenant existence leak, no TenantMismatch status.
- Run read delegates to the already tenant-scoped store query, so another tenant's or another run's rows are never returned.
- Unknown key / empty run -> NotFound. Reads use AsNoTracking and never write, never update timestamps, never touch AgentRun rows.

### ordering behavior

Run history is ordered by CreatedAtUtc ascending (oldest first), then ledger key ascending -- inherited unchanged from the store. Proven by a test that inserts the later record first and asserts time order, not insertion order.

### endpoint decision

No HTTP endpoint added. Service-level read only. A dev-only gated endpoint was considered and declined for this slice (unnecessary to prove the read surface; would widen the tenant-isolation surface). If added later it must be env-gated (development/admin), non-public, and tenant-scoped from caller identity, not a query parameter.

### what is intentionally not wired yet

- No production pipeline consumer; DI-registered (scoped) + resolution test only.
- No HTTP endpoint, no Angular, no merge execution, no artifact/AgentRun write, no confidence/posture/lean mutation, no analyze split.
- ProbeRefreshMergeAuditSummary / store-result metadata summarization deferred.

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 472 passed, 0 failed (includes +14 read-surface tests and +1 DI-resolution test). Cover read-by-key returns record; key read tenant-scoped (NotFound for another tenant); unknown key NotFound; missing tenant/idempotency InvalidInput; read-by-run returns run rows; run read excludes other tenant and other run; missing run id / missing tenant InvalidInput; run history CreatedAtUtc ascending (later row inserted first); read does not mutate audit rows; read does not update AgentRun rows; read service has no IToolGateway dependency. Existing audit store + contract tests stay green.
- Full runner normal pass (no "Build FAILED with 0 Error(s)" hang). No EF migration added; no PowerShell file changed. No Python/Angular change.

### risks

Low. Additive read-only facade + one scoped DI registration + tests; no schema change, no store change. Tenant isolation enforced by the post-fetch guard and the store's tenant-scoped run query; the no-TenantMismatch decision removes a cross-tenant leak vector. Reversible. Note: the idempotency read materializes one globally-unique row before the tenant guard; it is never returned cross-tenant, but a future hardening could push the tenant filter into the query.

### next slice

A future merge persistence/writer slice (still dormant, flagged) that consumes a ReadyForPersistCandidate audit record and projects an actual artifact write behind a feature flag, with rollback from the before-payload reference -- or a dev-only gated read endpoint over this service if an operator inspection surface is needed first. Keep confidence/posture/lean and artifact persistence untouched; do not split the analyze call.

### Claude/Codex transfer notes

- The read service is read-only and dormant; it reuses the store's queries and adds validation + a tenant guard. Do not add a write path or HTTP endpoint without a dedicated, gated slice.
- Idempotency reads are tenant-scoped at the service boundary (post-fetch guard) because the store method is keyed on the globally-unique key only. Keep that guard; a cross-tenant hit must remain NotFound (never TenantMismatch).
- Run history ordering is CreatedAtUtc ascending; preserve it if the store query changes.
- No migration was added; the read surface needs none.

### jera-workspace-skills status

Untouched (read-only this slice). Sharpening idea logged; not applied without approval.

status: Probe Refresh Merge Audit Read Surface v1 implemented 2026-06-05. Added IProbeRefreshMergeAuditReadService/ProbeRefreshMergeAuditReadService + ProbeRefreshMergeAuditReadResult/ProbeRefreshMergeAuditRunHistory/ProbeRefreshMergeAuditReadStatus in DevCore.Api.Protocols; tenant-safe read by idempotency key (post-fetch tenant guard, cross-tenant -> NotFound, no TenantMismatch) and by tenant+run (ordered CreatedAtUtc asc), reusing the store's queries. Read-only, no schema change, DI scoped, dormant, no HTTP endpoint. dotnet 472 (targeted 64). No merge/artifact/AgentRun write, no confidence/posture/lean mutation, no gateway/model call, no analyze split. jera-workspace-skills untouched.

## addendum: Deferred Runtime Decisions Ledger v1 (2026-06-05)

Docs-only slice. No runtime code, no tests, no schema/Angular/gateway change. Created a durable index of consciously deferred runtime decisions so deferral stays a choice, not drift.

### skills/guidance used

- superpowers: planning / writing-plans (entry schema designed before writing), verification-before-completion (git + path checks).
- Local jera-workspace-skills/dai (read-only, guidance applied manually): dai-grill-with-vault (located the right home + cross-refs), dai-token-tight (operational entries, not philosophical), dai-agent-handoff (this addendum). Pack not edited.

### files changed

dai-vault:
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- NEW. The ledger: one row per deferred decision with Decision / Current choice / Why deferred / Revisit trigger / Proposed future slice / Risk if forgotten / Status, plus how-to-use and maintenance notes.
- `06 Execution/handoffs/current-slice.md` -- this addendum.

dai: untouched. jera-workspace-skills: untouched.

### deferred decisions captured (12)

1. Interrogate requests; orchestrator (not Interrogate) triggers any Perceive refresh.
2. ProbeRefreshExecutor activation (default-disabled flag).
3. Probe-refresh artifact mutation / merge writer.
4. Confidence / posture / lean mutation (forbidden; protected-field guard).
5. Decide recommendation vs actual Decide mutation.
6. Synthesize preview vs user-facing final artifact.
7. Audit persistence vs artifact persistence.
8. Dev-only audit read endpoint (declined this slice).
9. pgvector / memory-backed probe.
10. Kubernetes / AKS / Azure Functions deployment.
11. Tenant / Stripe / economic boundary integration.
12. Calibration proof before posture-threshold changes.

### verification

- git status --short for dai-vault shows only the new ledger and this handoff.
- No exact local paths added (placeholders used in any path reference).
- Docs only; no .NET tests required or run.

### risks

Low. Documentation only. Main risk is staleness: the ledger must be updated when a slice resolves an entry or a handoff creates a new deferral. Maintenance note in the ledger records this.

### next slice

The deferred chain is fully indexed. The next runtime slice is the merge writer track (entry 3): a dormant, flagged Probe Refresh Merge Writer v1 that consumes a ReadyForPersistCandidate audit record and projects an artifact write behind a feature flag with rollback from the before-payload reference. Keep confidence/posture/lean and artifact persistence untouched until that slice explicitly scopes them.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Deferred Runtime Decisions Ledger v1 implemented 2026-06-05. Added deferred-runtime-decisions-ledger-v1.md in 02 Platform/architecture/cognitive-factory capturing 12 deferred runtime decisions (Decision/Current choice/Why deferred/Revisit trigger/Proposed future slice/Risk if forgotten/Status). Docs only; no runtime code, no tests, no schema/gateway/Angular change. Placeholders only, no exact local paths. jera-workspace-skills untouched.

## addendum: Probe Refresh Chain Assembly v1 (2026-06-05)

Assembly slice. Composes the dormant probe-refresh seams into one end-to-end orchestration object (ProbeRefreshChainAssembly). Proves the factory line can be assembled without mutating the artifact or wiring into production. Disabled by default, deterministic, no gateway call by default, no artifact mutation, no confidence/posture/lean change, no endpoint, no analyze split.

### pre-coding repo-state check

Verified clean and in sync before changes: <DAI_REPO_ROOT>, <DAI_VAULT_ROOT>, <JERA_SKILLS_ROOT>. Deferred Runtime Decisions Ledger v1 committed (dai-vault b7a181f). Probe Refresh Merge Audit Read Surface v1 committed (dai 3e86daf). All prior seams present and read before composing.

### skills/guidance used

- superpowers: planning / writing-plans (sequenced the 13-step orchestration + failure-status mapping against the real seam contracts), test-driven-development (fake executor + real in-memory store), systematic-debugging on standby (dev-host lock check; none running), verification-before-completion (fresh runs below).
- Local jera-workspace-skills/dai (read-only): dai-grill-with-vault (read every seam contract before composing -- surfaced that NotAuthorized is unreachable through the real seams), dai-token-tight, dai-agent-handoff. Pack not edited.
- Skill sharpening note: dai-grill-with-vault earned its keep again. No weakness found.

### naming decisions

- IProbeRefreshChainAssembly / ProbeRefreshChainAssembly; ProbeRefreshChainAssemblyRequest; ProbeRefreshChainAssemblyResult; ProbeRefreshChainAssemblyStatus; ProbeRefreshChainAssemblyOptions; ProbeRefreshChainStepResult (compact per-step trace); ProbeRefreshChainArtifactContext (matchup + source-artifact inputs).
- Added a safe-default MergeAuthority option (default ManualReviewRequired). Mutation flags (AllowArtifactMutation / AllowConfidenceMutation / AllowPostureMutation / AllowLeanMutation) are carried, default false, and are INERT in v1 (no writer reads them).
- Seams are optionally injectable (default to their static Default); DI passes only the required executor + store, while tests inject a fake seam to drive a specific branch.

### files changed

dai:
- platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainAssembly.cs -- NEW. Status enum, options/request/result/step/artifact-context records, interface + service composing decision -> authorization -> executor -> intake -> discern -> decide -> synthesize -> merge plan -> review -> dry-run -> audit -> (optional) store.
- platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs -- register IProbeRefreshChainAssembly scoped.
- platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshChainAssemblyTests.cs -- NEW. 13 tests (fake executor, in-memory store).
- platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs -- +1 resolution test.

No EF schema/migration change; no PowerShell change. No existing seam behavior changed.

dai-vault:
- 02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md -- added entry 13 (chain assembly activation deferral); entries 1-4 and 9-12 left unresolved.
- 06 Execution/handoffs/current-slice.md -- this addendum.

jera-workspace-skills: untouched.

### chain assembly contract summary

AssembleAsync(ProbeRefreshChainAssemblyRequest) -> ProbeRefreshChainAssemblyResult. Request carries ProbeRequest, CompetitionCode, ExistingRead, ArtifactContext (home/away/date + source artifact version/snapshot/id + existing field values), tenant/run/correlation/createdBy/createdAt, and Options. Result carries every seam output (decision, authorization result + authorized candidate, execution, intake, discern, decide, synthesize, merge plan, review, dry-run, audit record, store result), a per-step trace, status, reason, and tenant/run/correlation metadata.

### options/default behavior

Enabled=false, AllowGatewayExecution=false, PersistAuditRecord=false, MergeAuthority=ManualReviewRequired, all mutation flags false. Disabled by default; with default options the chain returns Disabled and calls no downstream seam.

### assembly behavior summary

Disabled when not enabled. NoProbeRequested when probe is null/NoProbeNeeded. NoRefreshWarranted when the decision warrants no refresh. NotAuthorized when no candidate authorizes at platform.retrieve (defensive; see below). ExecutionSkipped when AllowGatewayExecution=false (or the executor is disabled) -- the default enabled path stops here. With AllowGatewayExecution=true and a successful execution, the chain runs intake -> discern -> decide -> synthesize -> merge plan -> review -> dry-run -> audit, and persists the audit row only when PersistAuditRecord=true. Each failure stops with a specific status and a partial result; the step trace records how far it reached. The executor is always called at platform.retrieve, never at a cognitive station.

### how far the chain assembles

End to end through the audit record (status Assembled). With the default MergeAuthority=ManualReviewRequired: review=ManualReviewRequired, dry-run=ReviewNotPassed, audit=ReadyForManualReview -- the full line assembles conservatively with nothing auto-merging. With MergeAuthority=PlatformOnly: review passes, dry-run=Projected, audit=ReadyForPersistCandidate. NotAuthorized is a defensive branch: with the real decision+authorization a RefreshWarranted decision always authorizes at platform.retrieve, so it is unreachable through the real seams (tested via an injected fake authorization service).

### audit store behavior

PersistAuditRecord=false builds the audit record but writes nothing (asserted: zero rows). PersistAuditRecord=true uses the existing ProbeRefreshMergeAuditStore idempotency: first assembly Inserted, repeat assembly Existing, one row total. No DB migration; reuses the existing ProbeRefreshMergeAudits table.

### protected fields

Confidence, posture, lean, artifact version, tenant key, run id, raw retrieved signals, and historical audit deletion are forbidden mutations. The assembly only ever proposes preview metadata (RefreshMetadata / SynthesizePreview / QualityWarning / DecideRecommendation field paths), so no protected field is proposed or projected; the merge planner's forbidden-change list, the review forbidden-field guard, and the dry-run forbidden-field filter all remain in force. Tests assert no proposed/projected change touches a confidence/posture/lean path and that the supplied source-artifact snapshot is unchanged.

### what is intentionally not wired yet

- Disabled by default; no production pipeline consumer (DI-registered scoped + a resolution test only).
- No HTTP endpoint, no Angular, no artifact mutation, no merge writer, no confidence/posture/lean change, no analyze split.
- Gateway execution requires both the assembly option and an enabled executor; both default off. Mutation flags are inert.

### tests

- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 486 passed, 0 failed (was 472; +13 assembly tests, +1 DI-resolution test). Cover: disabled returns Disabled and calls nothing; NoProbeNeeded -> NoProbeRequested; no refresh warranted -> NoRefreshWarranted; injected no-auth -> NotAuthorized; default enabled run does not call the executor and records ExecutionSkipped; fake execution assembles through the audit record; plan/review/dry-run/audit produced; PlatformOnly authority passes review and projects the dry-run; persist=false writes nothing; persist=true writes once and is idempotent; tenant/run/correlation metadata carried; snapshot and protected fields never mutated; no http endpoint and no IToolGateway dependency. Existing audit read/store/dry-run/review/merge contract tests stay green.
- Full runner returned a normal pass (no "Build FAILED with 0 Error(s)" hang). No DevCore.Api.exe host running. No Python/Angular change.

### risks

Low-to-moderate. The assembly can call the gateway via the executor, but only when both the assembly option and the executor are enabled, both default off. It mutates nothing and persists only the audit row (idempotent) when asked. It reuses existing seams unchanged. The largest surface is the result object; that is read-only data. Reversible (delete the assembly + registration + tests). NotAuthorized is unreachable through real seams today -- kept as a defensive branch for future tools that do not list platform.retrieve.

### next slice

Probe Refresh Merge Writer v1 (ledger entry 3): a dormant, flagged writer that consumes a ReadyForPersistCandidate audit record and projects an actual artifact write behind a feature flag with rollback from the before-payload reference. Only after that should chain activation (ledger entry 13) be considered. Keep confidence/posture/lean and artifact persistence untouched until those slices scope them; do not split the analyze call.

### Claude/Codex transfer notes

- The assembly is dormant (disabled by default) and reuses the existing seams via their static Defaults (deterministic seams) plus the injected scoped executor + store. Do not enable it in production without the merge writer + activation slices.
- It authorizes and executes only at platform.retrieve, never at a cognitive station -- the standing boundary holds.
- Merge PLANNING is enabled inside the assembly to produce a dormant plan; this is not artifact mutation. The mutation flags are inert; do not use them to mutate anything until a writer exists.
- NotAuthorized is defensive; today every RefreshWarranted decision authorizes at platform.retrieve.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Probe Refresh Chain Assembly v1 implemented 2026-06-05. Added IProbeRefreshChainAssembly/ProbeRefreshChainAssembly + request/result/options/status/step/artifact-context records in DevCore.Api.Protocols; composes decision -> authorization -> executor -> intake -> discern -> decide -> synthesize -> merge plan -> review -> dry-run -> audit -> optional store into one dormant orchestration result. Disabled by default; gateway execution and audit persistence both opt-in; no artifact mutation, no confidence/posture/lean change, no endpoint, no schema change. dotnet 486 (targeted 64). Ledger entry 13 added; entries 1-4 and 9-12 unresolved. jera-workspace-skills untouched.

## addendum: Probe Refresh Chain Diagnostics v1 (2026-06-05)

Diagnostics-only slice. Adds a dormant service-level inspection surface for probe-refresh chain options and already-created assembly results. It does not run the chain, call the Tool Gateway, write the audit store, mutate artifacts, merge anything, change confidence/posture/lean, expose an endpoint, or wire into a production pipeline.

### pre-coding repo-state check

Verified before changes: <DAI_REPO_ROOT>, <DAI_VAULT_ROOT>, and <JERA_SKILLS_ROOT> were clean and in sync with origin. Probe Refresh Chain Assembly v1 was committed at dai 0304372 and dai-vault 9961a75.

### skills/guidance used

- superpowers: planning / writing-plans (naming and report shape before coding), test-driven-development (diagnostic behavior tests first against the new service surface), systematic-debugging (used after a direct filtered test returned no useful output), verification-before-completion (safe targeted/full runners + git/path checks).
- Local <JERA_SKILLS_ROOT> guidance applied manually: dai-grill-with-vault, dai-agent-handoff, dai-token-tight, dai-write-skill. Pack not edited.
- Skill sharpening note: the repo-hygiene/privacy pattern is now covered by the local placeholder convention; no skill change made.

### naming decisions

- Chosen: `IProbeRefreshChainDiagnostics`, `ProbeRefreshChainDiagnostics`, `ProbeRefreshChainDiagnosticReport`, `ProbeRefreshChainDiagnosticStatus`, `ProbeRefreshChainSafetySummary`, `ProbeRefreshChainStepDiagnostic`.
- Statuses: `SafeByDefault`, `EnabledButNonMutating`, `GatewayExecutionAllowed`, `AuditPersistenceAllowed`, `UnsafeMutationAllowed`, `ResultInspected`, `InvalidInput`.
- Avoided activation/executor/writer names because this slice only inspects options/results.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainDiagnostics.cs` -- NEW. Read-only diagnostics contract, report records, safety summary, step diagnostics, and implementation.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- registers `IProbeRefreshChainDiagnostics` as singleton.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshChainDiagnosticsTests.cs` -- NEW. 13 focused diagnostics tests.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- +1 DI resolution test.

dai-vault:
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- entry 13 status updated to note diagnostics shipped, activation still deferred.
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### diagnostics contract summary

- `DescribeOptions(ProbeRefreshChainAssemblyOptions?)` classifies safety posture from options only.
- `DescribeResult(ProbeRefreshChainAssemblyResult?)` summarizes an existing result only: final status, step counts, completed/skipped/failed steps, and whether gateway execution or audit-store action is present in the inspected result.
- `DescribeSafetyDefaults()` reports the default disabled, non-mutating posture.

### safety/default behavior

Default options report `SafeByDefault`: disabled, no gateway call, no audit persistence, no artifact mutation, no confidence/posture/lean mutation. `Enabled=true` with all execution/persistence/mutation flags off reports `EnabledButNonMutating`. `AllowGatewayExecution=true` and `PersistAuditRecord=true` are surfaced explicitly. Any mutation flag produces `UnsafeMutationAllowed` plus warnings.

### protected fields surfaced

Diagnostics always returns the protected field boundary: confidence, posture, lean, artifact-version, tenant, run-id, raw-retrieved-signals-overwrite, historical-audit-deletion.

### endpoint decision

No endpoint added. Service-level only. A diagnostics endpoint would be a separate gated slice; none was needed to satisfy this inspection surface.

### what is intentionally not wired yet

- No production pipeline consumer.
- No chain execution from diagnostics.
- No Tool Gateway call, no model/external call, no DB write, no audit-store write.
- No artifact writer, merge executor, confidence/posture/lean mutation, Angular work, FastAPI prompt change, analyze split, schema migration, MCP, pgvector, Azure Functions, Kubernetes, or secrets change.

### tests

- A direct filtered `dotnet test ... --filter ProbeRefreshChainDiagnosticsTests` returned exit code 1 with no useful output after a long wait, so verification switched to the repo safe runner.
- safe .NET targeted: 64 passed, 0 failed.
- safe .NET full: 500 passed, 0 failed. Covers new diagnostics tests plus existing chain assembly and audit read/store tests.
- No EF/database files changed. No PowerShell files changed, so no PowerShell parser validation required.

### deferred ledger updates

Updated ledger entry 13 status from assembly-only dormant to assembly + diagnostics shipped, dormant. No new deferral row added. Activation remains deferred behind the orchestrator trigger, executor activation, merge writer, and readiness decision.

### risks

Low. Additive pure service + singleton DI registration + tests. The report exposes whether options/results imply gateway or audit-store activity, but it does not cause either. Main residual risk is semantic drift if future activation adds fields not reflected in diagnostics; update diagnostics in the activation/readiness slice.

### next slice

Probe Refresh Chain Activation Readiness Review v1: use diagnostics to inventory remaining blockers and decide the exact activation preconditions before any writer or production wiring is attempted.

### Claude/Codex transfer notes

- Diagnostics inspect only supplied options/results. Do not call `IProbeRefreshChainAssembly` from diagnostics.
- Keep the protected-field list aligned with merge review/dry-run guards if those guards change.
- If a future endpoint is requested, keep it dev/admin gated and tenant-safe; do not make it public.
- Use placeholders in reports/docs and keep <JERA_SKILLS_ROOT> read-only unless explicitly approved.

### jera-workspace-skills status

Untouched.

status: Probe Refresh Chain Diagnostics v1 implemented 2026-06-05. Added IProbeRefreshChainDiagnostics/ProbeRefreshChainDiagnostics + diagnostic report/safety/step records in DevCore.Api.Protocols; singleton DI registration; 13 diagnostics tests plus DI resolution. Diagnostics only: no endpoint, no pipeline wiring, no gateway/model call, no DB write, no artifact merge/mutation, no confidence/posture/lean change, no schema change. dotnet 500 (targeted 64). Ledger entry 13 updated; activation remains deferred. jera-workspace-skills untouched.

## addendum: Probe Refresh Chain Activation Readiness Review v1 (2026-06-05)

Docs-only readiness review before any probe-refresh activation, writer, endpoint, or production wiring. No runtime code changed. No endpoint, chain execution, production pipeline consumer, artifact mutation, confidence/posture/lean mutation, Tool Gateway/model/external call, DB write, schema/FastAPI/Angular/MCP/pgvector/Azure/Kubernetes/secrets change.

### pre-change repo-state check

Verified before changes: <DAI_REPO_ROOT>, <DAI_VAULT_ROOT>, and <JERA_SKILLS_ROOT> were clean and in sync with origin. Probe Refresh Chain Diagnostics v1 had landed at dai f1a6671 and dai-vault 8db4843.

### skills/guidance used

- superpowers: planning / writing-plans (readiness sections and checklist), verification-before-completion (git status, doc/path checks), systematic-debugging on standby. No code change, so TDD did not apply.
- Local <JERA_SKILLS_ROOT> guidance applied manually and read-only: dai-grill-with-vault, dai-agent-handoff, dai-token-tight, dai-write-skill. Pack not edited.
- Skill sharpening note: no new weakness found. The existing placeholder convention kept local-path hygiene explicit.

### docs/code decision

Docs-only. The shipped `ProbeRefreshChainDiagnostics` already supplies the service-level inspection surface; adding a readiness value object would duplicate vault doctrine without changing runtime behavior. No code was added.

### files changed

dai: untouched.

dai-vault:
- `02 Platform/architecture/cognitive-factory/probe-refresh-chain-activation-readiness-v1.md` -- NEW. Activation readiness review and checklist.
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- entry 13 status updated to note readiness review complete while activation remains deferred.
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### naming decisions

- Document: `probe-refresh-chain-activation-readiness-v1.md`.
- Section names: Current dormant chain, Safe-by-default guarantees, Activation blockers, Required feature flags, Required telemetry/diagnostics, Tenant/run/idempotency boundaries, Protected fields, Audit/store/read requirements, Allowed future activation scope, Explicitly forbidden activation scope, Readiness checklist, Recommended next slices.
- Status vocabulary in the checklist: `ready` for shipped prerequisites, `blocked` for activation blockers. No activation/executor service names were introduced.

### readiness review summary

The review records that the chain is ready for planning a future limited dev activation, but not ready for runtime activation. Safe defaults, diagnostics, audit contract/store/read, idempotency, tenant/run boundaries, and protected-field guards are present. Activation remains blocked by missing production consumer, endpoint decision, config-bound feature flags, runtime telemetry emission, operator review surface, merge writer, rollback execution, tenant/Stripe economic boundary, and calibration proof for confidence/posture changes.

### safe-by-default guarantees captured

`Enabled=false`, `AllowGatewayExecution=false`, `PersistAuditRecord=false`, artifact mutation false, confidence/posture/lean mutation false, executor disabled by default, gateway constrained to `platform.retrieve`, diagnostics inspect only supplied options/results, and audit persistence remains ledger-only and opt-in.

### protected fields captured

confidence, posture, lean, artifact version, tenant, run id, raw retrieved signals overwrite, and historical audit deletion.

### explicitly forbidden activation scope

Direct Interrogate -> Perceive self-invocation, direct tool power on `interrogate.probe`, production activation without tenant/auth boundary, mutation without audit record, posture/confidence changes without calibration proof, public endpoint without explicit gate, artifact overwrite without rollback, audit store as artifact writer, gateway execution outside `platform.retrieve`, and unrelated FastAPI/Angular/schema/cloud/secrets changes.

### deferred ledger updates

Entry 13 remains Deferred. It now records Activation Readiness Review v1 as shipped and lists the remaining blockers: entries 1-3, feature-flag/config design, telemetry emission, operator review, tenant/economic gating, and product approval. No new ledger item was added.

### verification

Docs-only: no .NET tests required or run. Runtime files untouched. Schema/migration files untouched. New/added docs scanned for exact local path and local account-name patterns. ASCII check run against the new readiness doc.

### risks

Low. Documentation only. Main risk is staleness: if activation, telemetry, or writer slices change the boundaries, update the readiness doc and ledger in the same slice.

### next slice

Probe Refresh Limited Dev Activation Plan v1: design the config and tenant-scoped feature flags for dev/local activation only, still with no production wiring or artifact mutation.

### Claude/Codex transfer notes

- Do not start activation from this review alone. Treat it as the readiness gate.
- Keep chain activation separate from merge writer, telemetry emission, and operator review surface.
- Keep <JERA_SKILLS_ROOT> read-only unless explicitly approved.
- Use placeholders in reports/docs.

### jera-workspace-skills status

Untouched.

status: Probe Refresh Chain Activation Readiness Review v1 implemented 2026-06-05. Added a docs-only readiness review in dai-vault with safe defaults, blockers, required flags, telemetry/diagnostics needs, tenant/run/idempotency boundaries, protected fields, audit requirements, allowed limited activation scope, forbidden activation scope, checklist, and next slices. No runtime code, no tests required, no endpoint, no pipeline wiring, no DB/schema/FastAPI/Angular/cloud/secrets change. Ledger entry 13 updated but remains Deferred. jera-workspace-skills untouched.

## addendum: Factory Line Balance Review v1 (2026-06-06)

Docs-only architecture assessment. Maps the maturity of the whole cognitive factory line (Perceive, Interrogate, Discern, Decide, Synthesize) against the deeply-built `interrogate.probe` refresh path and recommends the next implementation slice. No runtime code, no endpoint, no chain execution, no artifact/confidence/posture/lean mutation, no Tool Gateway/model/external call, no DB write, no schema/FastAPI/Angular/MCP/pgvector/Azure/Kubernetes/secrets change.

### pre-change repo-state check

Verified clean before changes: <DAI_REPO_ROOT> (main), <DAI_VAULT_ROOT> (main), <JERA_SKILLS_ROOT> (main). Probe Refresh Chain Activation Readiness Review v1 confirmed committed in dai-vault (commit ecf5067).

### skills/guidance used

- Local <JERA_SKILLS_ROOT>/dai (read-only): dai-grill-with-vault (read code + vault before asserting, report disagreements not smoothed), dai-token-tight, dai-agent-handoff (this addendum shape). dai-write-skill not used -- no skill authored. Pack not edited.
- superpowers: planning / writing-plans (assessment structure), verification-before-completion (git status + path/ASCII checks). systematic-debugging on standby; no doc/code conflict surfaced.

### docs/code decision

Docs-only. The assessment is doctrine, not runtime surface. Grounded in reading the probe-refresh chain, `ProtocolNodeRunner`, `ProtocolStationCard`, `ProtocolStationDiagnostics`, `ProtocolToolAccessPolicy`, `ProtocolRegistry`, `CognitiveProtocolBuilder`, and the FastAPI analyzer seed. No code added.

### files changed

dai: untouched.

dai-vault:
- `02 Platform/architecture/cognitive-factory/factory-line-balance-review-v1.md` -- NEW. The factory-line balance review.
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- entry 14 added (genericization of probe-refresh-specific seams consciously deferred behind the recommended slice).
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### factory-line balance summary

Deep at one station, thin elsewhere. `interrogate.probe` carries a 17-seam dormant refresh chain + a one-station runner execute path; the other four macro protocols have no station-specific runtime seam beyond their slice of the single FastAPI analyze call (Synthesize is deterministic `SportsComposer`). Generic station infrastructure exists but is shallow: `ProtocolNodeRunner.ExecuteAsync` supports one station; `ProtocolStationCard` v0 encodes 10 of 18 blueprint fields (acknowledged in the card header, not a doc/code conflict); diagnostics and tool-access policy are read-only and dormant. Doctrine maturity (cards, ownership, boundaries) is uniformly high; runtime maturity is concentrated almost entirely in the dormant Interrogate loop.

### maturity findings by macro protocol

- Perceive: 3 model-emitted micro-actions; no generic intake. `ProbeRefreshPerceiveIntake` exists but is hard-keyed to sports `ToolIds.*`/context types -- not generic.
- Interrogate: probe deterministic and live; Question/Verify model-emitted; the entire refresh chain + runner execute path live here, dormant.
- Discern: 3 model-emitted; Weigh has a deterministic grade backbone; `ProbeRefreshDiscernReweigh` is probe-specific.
- Decide: model-emitted; Confidence/Position deterministic + clamped; `ProbeRefreshDecideRecommendation` is recommend-only and probe-specific. No generic Decide guard.
- Synthesize: fully deterministic `SportsComposer`; `ProbeRefreshSynthesizePreview` is preview-only, never user-facing.

### overbuilt areas

Probe-refresh merge half (6 seams guarding a writer that does not exist); per-seam result plumbing duplicated 15+ times; probe-only execute path on a generically-named runner.

### underbuilt areas

No generic Perceive signal-intake layer; no generic station result/diagnostic contract; runner drives one station; per-station tool enforcement is doctrine only (stage-level enforcement today); no generic Decide policy guard.

### reusable patterns discovered

Station result envelope `(Status, Reason, ErrorMessage?, IsX)`; chain step trace `(Step, Reached, Outcome)`; safe-by-default options record with inert dangerous flags; fail-closed type-erased payload recovery; read-only diagnostics over a dormant seam; forbidden-field guard enforced at multiple seams.

### what remains probe-refresh specific

The sports payload mapping inside `ProbeRefreshPerceiveIntake`; the DiscernReweigh/DecideRecommendation/SynthesizePreview *logic* (rule-of-three not met for the logic, only the result shape); the merge half.

### what should become generic platform infrastructure

The result envelope and step trace (harvest now, past rule-of-three); the read-only diagnostics shape; later a generic `PerceiveSignalIntake` (only with a second consumer) and a generic Decide policy guard.

### open questions

(1) factory maturity vs sports product polish -- the load-bearing fork; direction says factory, which the review follows. (2) generalize Perceive intake now or wait -- wait (one consumer). (3) generic StationResult envelope before more seams -- yes (the recommended slice). (4) Discern re-weigh generic or probe-specific -- probe-specific until a second use case. (5) minimum product-facing artifact improvement -- open, scopes the slice if (1) flips to product. (6) Decide policy guard now or with the merge writer -- defer with the writer unless a non-probe write path appears first.

### recommended next implementation slice

Primary: Generic Station Result Envelope v1 -- harvest the recurring result shape + step trace into one generic, tested envelope in DevCore.Api.Protocols and re-express existing seams against it with no behavior change. Activates nothing; strengthens the wider line; past rule-of-three so no speculative-abstraction risk.
Backup: Protocol Diagnostics Rollup v1 -- read-only generic diagnostics rollup unifying the fragmenting diagnostics/telemetry/step-trace surfaces. Chosen over Decide Policy Guard and Discern Station Runner Groundwork because those deepen probe-adjacent runtime before the shared contract exists.

### deferred ledger updates

Added entry 14: genericization of the probe-refresh-specific seams (PerceiveSignalIntake / DiscernReweigh / DecideRecommendation logic) is consciously deferred until the recommended Generic Station Result Envelope slice lands and a second use case justifies generalizing the logic. Entries 1-6 and 13 reaffirmed as Deferred; this review changed none of their Status. Probe-refresh activation and merge writer remain parked.

### verification

Docs-only: no .NET tests required or run. Runtime files untouched. Schema/migration files untouched. New review doc ASCII-checked (clean). New/added docs use placeholders; no exact local paths added. git status --short run for all three repos.

### risks

Low. Documentation only. Main risk is staleness: if the recommended envelope slice or any activation/writer slice lands, update this review and the ledger in the same slice. Secondary risk: the recommendation assumes the factory-maturity axis (open question 1); confirm with the founder before the slice starts.

### next recommended prompt

> Implement Generic Station Result Envelope v1 (docs + .NET refactor, no activation). Extract the recurring probe-refresh result shape (Status + Reason + ErrorMessage? + IsX) and the chain step trace (Step, Reached, Outcome) into one generic, tested envelope in DevCore.Api.Protocols; re-express the existing ProbeRefresh* seams against it with zero behavior change. No gateway call, no writer, no endpoint, no prompt/confidence/posture/schema/Angular change. Start with the Naming and Skills Gate and a clean-repo check.

### Claude/Codex transfer notes

- This is an assessment, not an activation. Do not start the merge writer or chain activation from it.
- The harvest target is the result *shape*, not the station *logic*. Do not generalize PerceiveIntake/DiscernReweigh/DecideRecommendation logic on one use case.
- Keep <JERA_SKILLS_ROOT> read-only unless explicitly approved. Use placeholders in reports/docs.

### jera-workspace-skills status

Untouched.

status: Factory Line Balance Review v1 implemented 2026-06-06. Added a docs-only cognitive-factory architecture assessment mapping macro-protocol/station maturity against the dormant interrogate.probe refresh chain, identifying the deep-at-one-station imbalance, harvestable patterns (result envelope, step trace), overbuilt/underbuilt areas, and reaffirming the activation/merge-writer/mutation deferrals. Primary recommendation: Generic Station Result Envelope v1; backup: Protocol Diagnostics Rollup v1. Ledger entry 14 added; entries 1-6, 13 reaffirmed Deferred. No runtime code, no tests required, no endpoint/pipeline/DB/schema/FastAPI/Angular/cloud/secrets change. jera-workspace-skills untouched.

## addendum: Generic Station Result Envelope v1 (2026-06-06)

Platform-contract slice. Harvests the result/step shape that recurs across the probe-refresh chain into generic, domain-agnostic types in DevCore.Api.Protocols, and re-expresses only the safest subset (the chain step trace) against them with zero behavior change. No production pipeline wiring, no artifact/confidence/posture/lean mutation, no Tool Gateway behavior change, no model call, no endpoint, no activation, no merge writer, no DB/schema/migration/FastAPI/Angular/MCP/pgvector/Functions/Kubernetes/secrets change.

### pre-change repo-state check

Verified clean before changes: <DAI_REPO_ROOT> (main), <DAI_VAULT_ROOT> (main), <JERA_SKILLS_ROOT> (main). Factory Line Balance Review v1 confirmed committed (dai-vault 6234e05).

### skills/guidance used

- superpowers: test-driven-development (the 13 new tests written against the brief's enumerated cases alongside the small contract types, then run via the safe runner), writing-plans / planning (designed the contract against the actually-repeated shapes before coding), verification-before-completion (all results below are fresh safe-runner runs). systematic-debugging not needed -- no failure surfaced. frontend-design not applicable.
- Local <JERA_SKILLS_ROOT>/dai (read-only, pack not edited): dai-grill-with-vault (inspected existing result shapes before designing, per "do not create a speculative abstraction beyond what is already repeated"), dai-token-tight, dai-agent-handoff (this addendum shape). dai-write-skill not used -- no skill authored.
- Available but not applicable: dai-signal-follow-up-diagnostics (about run signal coverage, not result envelopes).
- Skill sharpening note: no weakness found. dai-grill-with-vault's "read before designing" discipline directly prevented over-abstraction here -- the envelope was deliberately added as available infrastructure without retrofitting the 15 seam records, because reading them showed each carries its own status enum that must not be forced into one.

### naming review result

- `ProtocolResultEnvelope<TStatus>` (generic over the seam's own status enum) chosen over the non-generic `ProtocolResultEnvelope`: every seam has its own status enum, so generic keeps the domain status while adding shared classification. Rejected forcing one global status enum (brief rule).
- `ProtocolResultDisposition` enum { Success, Skipped, Blocked, Failure } as the shared coarse classification; the four IsX helpers are computed from it so they cannot drift. Chosen over four independent stored bools.
- Dropped the candidate `IsTerminal` helper: no existing seam uses a terminal/transient concept, so adding it would be speculative. Documented in the type.
- `ProtocolStepTrace` + `ProtocolStepOutcome` enum { Reached, Skipped, Failed, Blocked } for the per-step trace. Collapsed the candidate `Reason`/`Status` step fields into a single `Detail` (the human outcome text) plus `ErrorMessage`, to avoid an ambiguous double text field. `Reached` is a computed helper, not a stored bool.
- `ProbeRefreshChainTraceMapping.ToProtocolStepTrace(...)` extension as the single-sourced domain->generic mapper, kept in a domain file so the generic types carry no probe-refresh references (development-contract separation).
- Rejected `ProtocolResultMetadata` / `ProtocolResultFactory` as separate types: metadata is a plain optional dictionary field and construction is covered by the envelope's static factory methods; separate types would be ceremony.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProtocolResultEnvelope.cs` -- NEW. Generic `ProtocolResultEnvelope<TStatus>` + `ProtocolResultDisposition` + Success/Skipped/Blocked/Failure factories. No domain references.
- `platform/dotnet/DevCore.Api/Protocols/ProtocolStepTrace.cs` -- NEW. Generic `ProtocolStepTrace` + `ProtocolStepOutcome`. No domain references.
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainTraceMapping.cs` -- NEW. `ToProtocolStepTrace` extension; single-sources the reached/skipped/failed classification lifted from the diagnostics inline check.
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainDiagnostics.cs` -- MOD. `ToDiagnostic` now classifies through the generic trace instead of an inline substring check; removed the now-duplicate private `IsSkippedOutcome`. Output record and all values unchanged.
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshChainAssembly.cs` -- MOD. Added a computed `StepTraces` accessor on `ProbeRefreshChainAssemblyResult` (projects the stored `Steps` to generic traces; `Steps` itself unchanged).
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolResultEnvelopeTests.cs` -- NEW. 5 tests.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolStepTraceTests.cs` -- NEW. 8 tests.

dai-vault:
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- entry 14 status progressed.
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### generic envelope behavior

`ProtocolResultEnvelope<TStatus>` carries the seam's own status enum value, a `ProtocolResultDisposition` (Success/Skipped/Blocked/Failure), a `Reason`, an optional `ErrorMessage`, and optional `Metadata`. `IsSuccess/IsSkipped/IsBlocked/IsFailure` are computed from the disposition. Static factories (`Success`, `Skipped`, `Blocked`, `Failure`) construct each disposition so callers express intent. It is generic over any `struct, Enum` status, so a seam composes it without abandoning its domain statuses.

### step trace behavior

`ProtocolStepTrace(Step, Outcome, Detail?, ErrorMessage?)` with `ProtocolStepOutcome` Reached/Skipped/Failed/Blocked and computed `IsReached/IsSkipped/IsFailed/IsBlocked`. `ProbeRefreshChainTraceMapping.ToProtocolStepTrace` maps a chain step: reached -> Reached; not reached with an outcome containing "skipped"/"disabled" -> Skipped; otherwise -> Failed. Blocked is intentionally not produced from a chain step, preserving the existing diagnostics behavior that counts a blocked (not-reached) review step as failed.

### what was re-expressed

Only the chain step trace, the safest subset:
- `ProbeRefreshChainDiagnostics.ToDiagnostic` classifies via the generic trace; the `ProbeRefreshChainStepDiagnostic` output record and every value it produces are unchanged.
- `ProbeRefreshChainAssemblyResult` gains a computed `StepTraces` projection; the stored `Steps` field and the `ProbeRefreshChainStepResult` record are unchanged.

### compatibility behavior

Zero behavior change. No existing record field, constructor, or enum value was removed or renamed. The diagnostics classification is identical (same counts, same enum lumping of blocked-as-failed) -- the substring check was lifted, not altered. The 15 probe-refresh seam result records were intentionally NOT retrofitted onto the envelope: each keeps its own status enum and IsX helper. The envelope and trace are available infrastructure plus one re-expression, adopted further only when a second consumer justifies it.

### what intentionally remains domain-specific

The 15 `ProbeRefresh*` seam result records and their status enums; the probe-refresh-specific intake/re-weigh/recommendation logic; the merge half. Only the result/trace *shape* is generic, not any station *logic*.

### what is intentionally not wired yet

No seam result record was migrated to `ProtocolResultEnvelope<TStatus>` (would be a large surface change for no behavior gain on one consumer). No production pipeline, endpoint, gateway call, writer, or activation. The envelope is not yet consumed by any seam beyond being available; the trace has exactly two consumers (diagnostics + the assembly accessor).

### tests

- safe .NET targeted: 64 passed, 0 failed (also confirms the new types compile through the full build graph).
- safe .NET full: 513 passed, 0 failed (was 500 before this slice; +5 envelope tests, +8 step-trace tests). All existing probe-refresh chain, diagnostics, and assembly tests still green.
- new tests cover: envelope success/skipped/failure(with error)/blocked + metadata; step trace reached/skipped/failed/blocked; chain step mapping reached/skipped/failed; the assembly `StepTraces` accessor projects without mutating `Steps`.
- No Python change -> pytest not run. No Angular change -> Angular build not run. No PowerShell file changed -> no parser validation needed. No EF/schema/migration change.

### risks

Low. Additive generic types + one behavior-preserving classification lift + one computed accessor. The classification is now single-sourced (mapping), so diagnostics and the accessor cannot drift. Reversible (delete the three new files, revert the two edits, delete the two test files). Main residual risk is future scope creep: someone retrofits all 15 seams onto the envelope at once -- do that incrementally, behind tests, only as seams are touched for other reasons.

### next slice

Protocol Diagnostics Rollup v1 (the backup from the balance review): a read-only generic diagnostics rollup unifying the fragmenting diagnostics/telemetry/step-trace surfaces, now that the generic trace exists to roll up. Alternatively, adopt `ProtocolResultEnvelope<TStatus>` in one freshly-touched seam to validate the contract against a real second consumer before broader rollout.

### Claude/Codex transfer notes

- The envelope is available infrastructure, not yet a required base type. Do NOT mass-migrate the 15 seam results onto it; adopt incrementally behind tests as seams are touched.
- The reached/skipped/failed classification is single-sourced in `ProbeRefreshChainTraceMapping`. Change it there, not inline in diagnostics.
- Blocked is deliberately absent from the chain mapping to preserve existing diagnostics counts; do not "fix" it without updating the diagnostics tests intentionally.
- Keep <JERA_SKILLS_ROOT> read-only unless explicitly approved. Use placeholders in reports/docs.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Generic Station Result Envelope v1 implemented 2026-06-06. Added generic `ProtocolResultEnvelope<TStatus>` + `ProtocolResultDisposition` and `ProtocolStepTrace` + `ProtocolStepOutcome` in DevCore.Api.Protocols, plus `ProbeRefreshChainTraceMapping`. Re-expressed only the chain step trace: diagnostics classify through the generic trace (output unchanged) and `ProbeRefreshChainAssemblyResult` exposes a computed `StepTraces` projection. Zero behavior change; the 15 seam result records were not retrofitted. dotnet 513 (targeted 64), +13 new tests. No model/prompt/gateway/confidence/posture/artifact/endpoint/schema/migration/Angular/MCP change. Ledger entry 14 progressed (generic shape landed; broader genericization still deferred pending a second consumer). jera-workspace-skills untouched.

## addendum: Protocol Diagnostics Rollup v1 (2026-06-06)

Read-only inspection slice. Adds a protocol diagnostics rollup that summarizes the protocol runtime's health and maturity WITHOUT executing anything, composing the existing inspection surfaces rather than duplicating their rules. No execution, no Tool Gateway call, no model call, no DB write, no artifact/confidence/posture/lean mutation, no production pipeline wiring, no endpoint, no activation, no merge writer. Behavior-preserving: no existing diagnostics behavior changed.

### pre-change repo-state check

Verified clean before changes: <DAI_REPO_ROOT> (main), <DAI_VAULT_ROOT> (main), <JERA_SKILLS_ROOT> (main). Generic Station Result Envelope v1 confirmed committed (dai 4275083, dai-vault 5316895).

### skills/guidance used

- superpowers: test-driven-development (13 tests against the brief's enumerated cases written with the contract, then run via the safe runner), writing-plans / planning (mapped the rollup onto existing surfaces before coding so it composes, not duplicates), verification-before-completion (all results below are fresh safe-runner runs). systematic-debugging not needed -- no failure surfaced. frontend-design not applicable (no endpoint, no UI).
- Local <JERA_SKILLS_ROOT>/dai (read-only, pack not edited): dai-grill-with-vault (inspected ProtocolStationDiagnostics, ProtocolRegistryValidator, ProtocolNodeRunner, ProbeRefreshChainDiagnostics before designing, per "inspect existing diagnostics before adding anything" + "prefer composing existing diagnostics over duplicating rules"), dai-token-tight, dai-agent-handoff (this addendum). dai-write-skill not used.
- Available but not applicable: dai-signal-follow-up-diagnostics (run signal coverage, not runtime rollup).
- Skill sharpening note: no weakness found. dai-grill-with-vault's read-first discipline is exactly why the card<->tool finding rule is composed from the existing ProtocolRegistryValidator (single-card validation) instead of being re-implemented in the rollup -- the rule cannot drift.

### naming review result

- `ProtocolDiagnosticsRollup` + `IProtocolDiagnosticsRollup` chosen over `ProtocolStationDiagnosticRollup`: the rollup spans runtime + stations + chain result, not stations alone.
- Methods: `DescribeRuntime()`, `DescribeStation(string stationId)`, `DescribeChainResult(ProbeRefreshChainAssemblyResult?)`.
- `ProtocolDiagnosticsRollupReport` (runtime), `ProtocolStationRollup` (per-station), `ProtocolChainResultRollup` (optional chain-result), `ProtocolRuntimeDiagnosticFinding`.
- `ProtocolDiagnosticsRollupStatus` { Healthy, Warnings, Blocked, InvalidInput }; `ProtocolDiagnosticSeverity` { Info, Warning, Error }.
- Used `MicroAction` (the existing card field name) rather than the brief's candidate `MicroProtocol`, to stay consistent with `ProtocolStationCard` / `ProtocolStationDiagnostic`.
- Rejected a `ProtocolDiagnosticsRollupSummary` separate type: the report record already is the summary.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProtocolDiagnosticsRollup.cs` -- NEW. The rollup service + interface + report/station/chain-result records + finding record + status/severity enums + a static `Default` wired to the canonical manifests.
- `platform/dotnet/DevCore.Api/Tools/ToolGatewayServiceCollectionExtensions.cs` -- MOD. Register `IProtocolDiagnosticsRollup` singleton wired to the existing singletons (no Program.cs change).
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProtocolDiagnosticsRollupTests.cs` -- NEW. 12 tests.
- `platform/dotnet/DevCore.Api.Tests/Tools/ToolGatewayDIRegistrationTests.cs` -- MOD. +1 test resolving `IProtocolDiagnosticsRollup` through the real Program.cs graph.

dai-vault:
- `06 Execution/handoffs/current-slice.md` -- this addendum.
- deferred-runtime-decisions-ledger-v1.md -- intentionally NOT changed (this slice changes no deferral; entry 14 is about genericizing seam logic, which this rollup does not touch).

jera-workspace-skills: untouched.

### diagnostics rollup behavior

`DescribeRuntime()` enumerates the 15 registry station cards and returns a report with: overall `Status` (Healthy/Warnings/Blocked), `StationCount`, `KnownStationCount`, `SupportedExecutionStationCount`, `ToolAwareStationCount`, `DiagnosticFindingCount`, the aggregated `Findings`, the per-station `Stations` rows, and `ProtectedFields` (surfaced from `ProbeRefreshChainDiagnostics.DescribeSafetyDefaults().Safety`). Findings are aggregated from per-station manifest cross-checks; a consistent manifest yields none -> Healthy.

### station rollup behavior

`DescribeStation(stationId)` returns a `ProtocolStationRollup`: `IsKnown`, `HasStationCard`, `HasRunnerSupport`, `HasToolPolicy`, `AllowedToolCount`, `ModelCallRule`, `CostClass`, `QualityGateCount`, `FindingCount`, `Findings`. It composes `ProtocolStationDiagnostics.DescribeStation` for the snapshot and `ProtocolRegistryValidator.Validate([card], tools)` for findings, so neither rule is re-implemented.

### chain-result rollup behavior

`DescribeChainResult(result)` summarizes the result's generic `StepTraces` (from Generic Station Result Envelope v1) into Reached/Skipped/Failed/Blocked counts. A trace with a failed or blocked step -> Blocked; an all-reached/skipped trace -> Healthy (an intentional skip is not unhealthy, mirroring ProbeRefreshChainDiagnostics). Null input -> InvalidInput. It inspects an existing result; it never replays the chain.

### unsupported station behavior

Runner execution support is reported, not exercised: only `interrogate.probe` has a deterministic `ExecuteAsync` path today (mirrored as a static supported-id set, so the rollup does NOT call `ExecuteAsync`). All other stations report `HasRunnerSupport = false`, which is NOT a finding and leaves the runtime Healthy. Stations with no allowed tools (the deterministic stations) report `HasToolPolicy = false` and are likewise not failures.

### what is intentionally not wired yet

No HTTP endpoint. No production pipeline path calls the rollup. No seam result was migrated onto `ProtocolResultEnvelope<TStatus>` (that remains the deferred broader-genericization work). The rollup is dormant, available infrastructure for a future dev/admin inspection caller.

### tests

- safe .NET targeted: 64 passed, 0 failed (also confirms the new code compiles through the full build graph).
- safe .NET full: 526 passed, 0 failed (was 513; +12 rollup tests, +1 DI-resolution test). All existing ProtocolStationDiagnostics, ProbeRefreshChainDiagnostics, and Generic Station Result Envelope tests still green.
- new tests cover: runtime counts/known stations; station rollups included; interrogate.probe runner-supported + SupportedExecutionStationCount=1; unsupported stations not failing; protected fields surfaced; known station IsKnown=true; unknown station IsKnown=false (Info finding); tool-aware allowed-tool counts; an injected unregistered-tool card -> Error finding + Blocked; chain-result summary of step outcomes; null chain result -> InvalidInput; failed chain step -> Blocked.
- No Python/Angular/PowerShell/EF/schema change -> those validations not applicable.

### deferred ledger updates

None. This slice changes no deferral. Entry 14 (genericizing the probe-specific intake/re-weigh/recommendation logic) is unaffected -- the rollup composes diagnostics; it does not generalize station logic. Activation, merge writer, and artifact/confidence/posture/lean mutation remain deferred.

### risks

Low. Additive read-only service + one DI registration + tests; no existing behavior changed. Findings and protected fields are composed from existing single-sourced surfaces (ProtocolRegistryValidator, ProbeRefreshChainDiagnostics), so they cannot drift. Reversible (delete the new file + the one registration + tests, revert the DI test). Residual: the runner-supported station set is mirrored as a constant -- if the runner gains a second deterministic station, update both (a comment points to ProtocolNodeRunner).

### next slice

Either adopt `ProtocolResultEnvelope<TStatus>` in one freshly-touched seam to validate the contract against a real second consumer (progressing ledger entry 14), or a dev/admin-only, env-gated read-only inspection endpoint over `IProtocolDiagnosticsRollup` (mirroring the previously-declined audit-read-endpoint gating, ledger entry 8) if an operator need appears.

### Claude/Codex transfer notes

- The rollup is read-only and dormant; do not expect any runtime/pipeline change and do not add an HTTP endpoint without a dedicated, env-gated slice.
- Findings rules are single-sourced: the card<->tool check comes from ProtocolRegistryValidator, protected fields from ProbeRefreshChainDiagnostics. Add new findings by composing existing surfaces, not by re-implementing rules in the rollup.
- Runner execution support is a mirrored constant (interrogate.probe). Keep it in sync with ProtocolNodeRunner.ExecuteAsync; do not call ExecuteAsync from the rollup (it must not execute).
- Keep <JERA_SKILLS_ROOT> read-only unless explicitly approved. Use placeholders in reports/docs.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Protocol Diagnostics Rollup v1 implemented 2026-06-06. Added IProtocolDiagnosticsRollup/ProtocolDiagnosticsRollup in DevCore.Api.Protocols: a read-only DescribeRuntime/DescribeStation/DescribeChainResult rollup composing ProtocolRegistry, ProtocolStationDiagnostics, ProtocolRegistryValidator, ProbeRefreshChainDiagnostics, and the generic ProtocolStepTrace. Reports station counts, runner support (interrogate.probe only, not executed), tool-awareness, protected fields, and manifest-mismatch findings; runs nothing. DI-registered, no endpoint, no pipeline path calls it. dotnet 526 (targeted 64), +13 tests. No model/prompt/gateway/confidence/posture/artifact/schema/migration/Angular/MCP change. Ledger unchanged (no deferral changed). jera-workspace-skills untouched.

## addendum: Probe Refresh Merge Review Envelope Adoption v1 (2026-06-06)

Envelope-adoption slice. Adopts the generic `ProtocolResultEnvelope<TStatus>` (Generic Station Result Envelope v1) in exactly ONE real probe-refresh seam -- merge review -- to prove the contract against production-shaped code without mass migration. Behavior-preserving: the domain result, its status enum, and all domain fields stay intact; the envelope is an additive computed projection. No activation, no endpoint, no Tool Gateway/model/external call, no DB/schema change, no FastAPI/Angular change, no artifact/confidence/posture/lean mutation, no merge writer.

### pre-change repo-state check

Verified clean before changes: <DAI_REPO_ROOT> (main), <DAI_VAULT_ROOT> (main), <JERA_SKILLS_ROOT> (main). Generic Station Result Envelope v1 confirmed committed (dai 4275083); Protocol Diagnostics Rollup v1 also landed (dai c36fb20) and was inspected -- not depended on by this slice.

### skills/guidance used

- superpowers: test-driven-development (13 mapping/preservation tests written with the projection, then run via the safe runner), writing-plans / planning (mapped the status->disposition table and chose conservative dispositions before coding), verification-before-completion (fresh safe-runner runs below). systematic-debugging not needed.
- Local <JERA_SKILLS_ROOT>/dai (read-only, pack not edited): dai-grill-with-vault (read the merge review seam + its tests + the envelope before mapping, per "if ProbeRefreshMergeReviewResult is not a clean candidate, stop and report" -- it was clean: read-only, governance-oriented, no fetch/persist/mutation), dai-token-tight, dai-agent-handoff. dai-write-skill not used.
- Available but not applicable: dai-signal-follow-up-diagnostics.
- Skill sharpening note: no weakness found. The read-first discipline confirmed merge review was the right first adopter (pure governance result, statuses map cleanly to dispositions) before any code was written.

### naming review result

- `ToEnvelope()` method chosen over an `Envelope { get; }` property: it constructs a projection via a switch, and a method reads as work/construction (matching the codebase's use of computed properties only for trivial booleans like `IsReviewPassed`). Rejected `AsEnvelope` / `ToProtocolResultEnvelope()` (verbose) and a separate `ProbeRefreshMergeReviewEnvelopeMapper` type (a local method keeps the mapping next to the result it projects).
- Private `DispositionFor(status)` switch as the single mapping point, with a conservative `_ => Failure` default for any future status.

### files changed

dai:
- `platform/dotnet/DevCore.Api/Protocols/ProbeRefreshMergeReview.cs` -- MOD. Added `ToEnvelope()` + private `DispositionFor` on `ProbeRefreshMergeReviewResult`. No field removed/renamed; the status enum and all domain fields are unchanged.
- `platform/dotnet/DevCore.Api.Tests/Protocols/ProbeRefreshMergeReviewEnvelopeTests.cs` -- NEW. 13 tests (7 status->disposition theory cases + status/reason/error preservation + two real-reviewer-driven cases).

dai-vault:
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- entry 14 progressed again (first real seam adopted the envelope).
- `06 Execution/handoffs/current-slice.md` -- this addendum.

jera-workspace-skills: untouched.

### envelope adoption summary

`ProbeRefreshMergeReviewResult.ToEnvelope()` returns `ProtocolResultEnvelope<ProbeRefreshMergeReviewStatus>` carrying the original domain `Status`, the mapped `Disposition`, the existing `Reason`, and the existing `ErrorMessage`. It is computed on access; the result record's shape and every domain field are unchanged.

### status-to-disposition mapping

- ReviewPassed -> Success (review cleared the plan).
- ManualReviewRequired -> Blocked (a governance hold/human gate, not an error).
- ReviewBlocked -> Blocked (policy blocked, e.g. a forbidden proposed change).
- UnsafePlan -> Blocked (recognized plan whose authority/state cannot pass automatic review; the review ran and said no -- a governance block, not a crash).
- Disabled -> Skipped (merge off; review intentionally bypassed).
- NoMergePlan -> Skipped (no plan supplied: an expected non-ready state, distinct from the malformed-input case, so not a Failure).
- InvalidInput -> Failure (plan input malformed before review could decide).

### compatibility behavior

Zero behavior change. No field, constructor, or enum value removed or renamed. `MayMerge`, `RequiresManualReview`, `BlockedReasons`, `TelemetryEvent`, and `IsReviewPassed` all remain. Existing merge review tests are untouched and still pass; the envelope tests are additive.

### why only one seam was adopted

The slice brief scopes adoption to exactly one seam to validate the contract against real, production-shaped code before any broad rollout. Merge review was chosen because it is read-only, governance-oriented, and does not fetch, persist, or mutate -- so its statuses map cleanly to dispositions with no behavioral risk. Migrating the other 14 seams now would be a large surface change for no behavior gain and would re-open the genericization the balance review deliberately staged.

### what intentionally remains domain-specific

The `ProbeRefreshMergeReviewStatus` enum, the `ProbeRefreshMergeReviewReason` taxonomy, `MayMerge` / `RequiresManualReview` / `BlockedReasons` / `TelemetryEvent`, and the review decision logic. The other 14 probe-refresh seam result records keep their own shapes and were not retrofitted.

### what is intentionally not wired yet

No other seam adopts the envelope. No consumer reads `ToEnvelope()` on a pipeline path; ProtocolDiagnosticsRollup was deliberately NOT changed to consume it (would widen the slice). The projection is available for future cross-seam tooling.

### tests

- safe .NET targeted: 64 passed, 0 failed (also confirms compilation through the full build graph).
- safe .NET full: 539 passed, 0 failed (was 526; +13). All existing ProbeRefreshMergeReview, ProtocolResultEnvelope, and ProtocolDiagnosticsRollup tests still green.
- new tests cover: all 7 status->disposition mappings; original status preserved; Reason carried; ErrorMessage carried when present and null when absent; two real reviewer outputs (ReviewPassed->Success, ManualReviewRequired->Blocked) with domain fields intact.
- No Python/Angular/PowerShell/EF/schema change -> those validations not applicable.

### risks

Low. Additive computed projection + tests on one seam; nothing else changed. The mapping is a pure function of Status with a conservative default. Reversible (delete the method + the test file). Residual: the disposition mapping is a judgement call (e.g. UnsafePlan -> Blocked vs Failure); it is documented in code and here so a future reviewer can revisit it deliberately.

### next slice

Either adopt the envelope in a second, different-shaped seam (e.g. ProbeRefreshExecutionResult or ProbeRefreshPerceiveIntakeResult) to further validate the contract across dispositions, or pause envelope adoption here as proven and pick a different factory-line slice. Do not mass-migrate; continue one seam at a time, behind tests.

### Claude/Codex transfer notes

- This is one-seam adoption, not a migration. Do NOT retrofit the other 14 seam results in one go.
- The status->disposition mapping is documented in ProbeRefreshMergeReview.cs; change it there with intent (it is a governance judgement, not mechanical).
- ToEnvelope() is additive and unconsumed on any pipeline path; do not wire it into runtime behavior without a dedicated slice.
- Keep <JERA_SKILLS_ROOT> read-only unless explicitly approved. Use placeholders in reports/docs.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Probe Refresh Merge Review Envelope Adoption v1 implemented 2026-06-06. Added ProbeRefreshMergeReviewResult.ToEnvelope() projecting the domain status to ProtocolResultDisposition (ReviewPassed->Success; ManualReviewRequired/ReviewBlocked/UnsafePlan->Blocked; Disabled/NoMergePlan->Skipped; InvalidInput->Failure) while preserving every domain field. One seam adopted, not a migration; the other 14 seam results and the probe-specific station logic are unchanged. dotnet 539 (targeted 64), +13 tests. No model/prompt/gateway/confidence/posture/artifact/endpoint/schema/migration/Angular/MCP change. Ledger entry 14 progressed again (first real seam adopted the envelope; broader genericization still deferred). jera-workspace-skills untouched.

## addendum: Perceive Signal Intake Standardization v1 (2026-06-06)

Contract slice. Adds a small generic Perceive signal intake contract that normalizes signal observations from analyzer seeds, platform refreshes, tools, manual input, and future memory sources. Behavior-preserving: no production pipeline wiring, no endpoint, no model call, no Tool Gateway call, no FastAPI prompt change, no Angular change, no DB/schema change, no artifact mutation, no merge writer, and no confidence/posture/lean mutation.

### pre-change repo-state check

Verified clean before changes: <DAI_REPO_ROOT> (main), <DAI_VAULT_ROOT> (main), <JERA_SKILLS_ROOT> (main). Recent commits confirmed present: dai `9dbc5d7104d5870cca3da8249127a2316ecd937c`; dai-vault `50fb1620b00e9feb2d6f1b45a595e2cfe761c62f`. <JERA_SKILLS_ROOT> was read-only and remained untouched.

### skills/guidance used

- Local <JERA_SKILLS_ROOT>/dai (read-only): dai-grill-with-vault (read Perceive/probe-refresh code and vault docs before design), dai-token-tight, dai-agent-handoff. dai-write-skill was inspected for boundary/doc-writing discipline but not used to create or edit skills.
- superpowers-style guidance applied manually: planning / writing-plans, test-driven-development, systematic-debugging, verification-before-completion.
- Naming and Skills Gate completed before coding: skills inspected, repo states checked, recent commits verified, names reviewed, and runtime cognitive protocols kept separate from development skills.

### naming review result

Chosen: `PerceiveSignalIntake`, `IPerceiveSignalIntake`, `PerceiveSignalObservation`, `PerceiveSignalSource`, `PerceiveSignalOrigin`, `PerceiveSignalContext`, `PerceiveSignalIntakeStatus`, `PerceiveSignalIntakeResult`.

Rejected for v1: `PerceiveSignalEvidence` and confidence/evidence-strength vocabulary. Existing behavior only needs `Summary` plus `HasSignalContext`; adding strength/confidence terms would imply confidence behavior this slice forbids.

### files changed

dai:
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/PerceiveSignalIntake.cs` -- NEW. Generic source/origin/context/observation/result contract, validation intake, and envelope projection.
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/Protocols/ProbeRefreshPerceiveSignalProjection.cs` -- NEW. One-way projection from `PerceiveRefreshView` / `ProbeRefreshPerceiveIntakeResult` into the generic contract.
- `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api.Tests/Protocols/PerceiveSignalIntakeTests.cs` -- NEW. 12 tests.

dai-vault:
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` -- entry 14 progressed; new entry 15 added for Perceive signal intake adoption.
- `<DAI_VAULT_ROOT>/06 Execution/handoffs/current-slice.md` -- this addendum.

<JERA_SKILLS_ROOT>: untouched.

### contract summary

`PerceiveSignalObservation` is the normalized input: `SignalKey`, `SourceKind`, `Origin`, `Summary`, `Context`, `HasSignalContext`, and optional `Metadata`. `PerceiveSignalContext` carries optional tenant/run/correlation/source-tool/context metadata: `TenantKey`, `RunId`, `CorrelationId`, `SourceToolId`, `ContextType`, `RawContextType`, `ObservedAtUtc`, and `Metadata`.

`PerceiveSignalIntake.Receive(...)` validates only the generic shape: null observation -> `NoSignal`; blank signal key -> `InvalidInput`; blank summary -> `NoSignal`; otherwise trims key/summary and returns `Received`. `PerceiveSignalIntakeResult.ToEnvelope()` maps Received->Success, NoSignal/UnsupportedSignal->Skipped, InvalidInput->Failure.

### source/origin model

`PerceiveSignalSource`: `AnalyzerSeed`, `PlatformRefresh`, `ManualInput`, `HistoricalMemory`, `Unknown`.

`PerceiveSignalOrigin`: `ModelEmitted`, `PlatformDerived`, `ToolFetched`, `UserProvided`, `Unknown`.

Analyzer seed signals are represented as `AnalyzerSeed` + `ModelEmitted`. Platform refreshes can be `PlatformRefresh` + `PlatformDerived`; fetched tool context is `PlatformRefresh` + `ToolFetched`. Manual placeholders are `ManualInput` + `UserProvided`. Memory is named but not wired.

### projection behavior

`PerceiveRefreshView.ToPerceiveSignalObservation(...)` maps:
- `SignalKey` -> `SignalKey`
- `ToolId` -> `Context.SourceToolId`
- `ContextType` -> `Context.ContextType`
- `PerceivedSummary` -> `Summary`
- `HasContext` -> `HasSignalContext`
- source/origin -> `PlatformRefresh` / `ToolFetched`

`ProbeRefreshPerceiveIntakeResult.ToPerceiveSignalIntakeResult(...)` preserves `RawContextType` and fails closed: NoContext->NoSignal, UnsupportedContext->UnsupportedSignal, InvalidPayload->InvalidInput. The projection is one-way, additive, and unconsumed by the chain.

### compatibility behavior

Zero runtime behavior change. `ProbeRefreshPerceiveIntakeResult` and `PerceiveRefreshView` remain intact. No constructor/enum/field was removed or renamed. `CognitiveProtocol`, `PerceiveProtocol`, FastAPI analyzer seed parsing, Tool Gateway behavior, DB schema, Angular, and production artifact merge/write behavior are unchanged.

### what remains probe-refresh-specific

The sports `ToolIds.*` switch, typed payload recovery, and deterministic context summaries stay in `ProbeRefreshPerceiveIntake`. `ProbeRefreshDiscernReweigh`, `ProbeRefreshDecideRecommendation`, `ProbeRefreshSynthesizePreview`, merge planning/review/dry-run/audit, and the chain assembly remain probe-refresh-specific and dormant.

### what is intentionally not wired yet

No production caller consumes `PerceiveSignalIntake`. Analyzer seed observations are not routed through it. Probe-refresh still uses its own result in the chain. No direct Interrogate -> Perceive loop, executor activation, merge writer, artifact mutation, confidence/posture/lean mutation, endpoint, schema, prompt, Angular, MCP, pgvector, Kubernetes/AKS, or tenant/Stripe change landed.

### tests

- `<DAI_REPO_ROOT>/scripts/dev/dotnet/test-devcore-api-safe.ps1 -Targeted` -- pass: 64 passed, 0 failed.
- direct filtered diagnostic run: `dotnet test <DAI_REPO_ROOT>/platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj --no-restore -v minimal --filter FullyQualifiedName~PerceiveSignalIntakeTests` returned exit 1 with no output after a long wait; no leftover `dotnet`/`testhost` processes were found. The safe full run below covered the new tests successfully.
- `<DAI_REPO_ROOT>/scripts/dev/dotnet/test-devcore-api-safe.ps1 -Full` -- pass: 551 passed, 0 failed (was 539; +12).

New tests cover analyzer seed observation, platform refresh observation, tool-fetched observation, manual/user-provided placeholder, missing signal key -> InvalidInput, empty summary -> NoSignal, UnsupportedSignal envelope mapping, projection preserving signal key/tool/context type/raw type, projection immutability, non-received probe-refresh fail-closed mapping, and Received envelope metadata.

### deferred ledger updates

Entry 14 progressed: the generic Perceive signal intake contract/projection shipped, but probe-refresh logic and other seam logic are not generalized. New entry 15 added: Perceive signal intake adoption across analyzer seed and refreshed context remains Deferred. Direct Interrogate -> Perceive refresh, activation, merge writer, artifact mutation, confidence/posture/lean mutation, memory/pgvector, Kubernetes/AKS, tenant/Stripe, and calibration threshold changes remain deferred.

### risks

Low. Additive value objects, a stateless validator, one-way projection, and tests. Residual risk is adoption ambiguity: without a follow-up adoption slice, future callers could still bypass the generic shape. No behavior changes means no immediate runtime risk.

### next slice

Perceive Signal Intake Adoption v1: wire read-only projections from analyzer seed and probe-refresh into the generic contract without changing model prompts, call counts, Tool Gateway behavior, artifacts, confidence/posture/lean, endpoints, schema, or Angular. Keep it projection-only unless a separate activation slice approves runtime use.

### Claude/Codex transfer notes

- Treat `PerceiveSignalIntake` as dormant infrastructure. Do not replace `ProbeRefreshPerceiveIntakeResult` or retrofit the chain without a dedicated adoption slice.
- Keep source/origin separate: source says where the observation enters from; origin says who/what produced it.
- Do not add confidence/evidence-strength fields unless a calibration-safe behavior slice explicitly needs them.
- Keep <JERA_SKILLS_ROOT> read-only unless explicitly approved. Use placeholders in reports/docs.

### jera-workspace-skills status

Untouched (read-only this slice).

status: Perceive Signal Intake Standardization v1 implemented 2026-06-06. Added generic Perceive signal observation/context/source/origin/intake/result contracts plus one-way probe-refresh projection. No production wiring, no model/prompt/gateway/confidence/posture/artifact/endpoint/schema/migration/Angular/MCP change. dotnet 551 (targeted 64), +12 tests. Ledger entry 14 progressed and entry 15 added for deferred adoption. jera-workspace-skills untouched.
