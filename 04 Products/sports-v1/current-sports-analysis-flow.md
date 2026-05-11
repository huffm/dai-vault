# current sports analysis flow

Last updated: 2026-05-09 (signal quality model v1)
Reflects code state after signal quality model v1 implementation.

---

## overview

A sports.matchup.analysis run executes four pipeline stages against a single shared working artifact.
Each stage reads its required inputs from the artifact and records its output and status back.
AgentRunService is the thin entry orchestrator -- it creates the artifact and calls the four stages in order.
There is exactly one model call per run, in the analyze stage.

The artifact carries publishability, degradation notes, and per-stage step results.
These are persisted in OutputJson for observability and the future learning loop.
The cognitive artifact v1 adds a compact 4-phase reasoning object to OutputJson and exposes only
delivery fields to the UI: Read Stance, Counter Case, Watch For, What Would Change the Read, and
nullable evidence richness.

---

## key types

| type | file | role |
|---|---|---|
| `SportsRunArtifact` | SportsRunArtifact.cs | working artifact; flows through all stages |
| `PipelineStepResult` | PipelineModels.cs | recorded outcome of one stage |
| `PipelineStepStatus` | PipelineModels.cs | Succeeded / Degraded / Skipped / Failed |
| `PublishabilityStatus` | PipelineModels.cs | Publishable / PublishableWithCaveats / NotPublishable |
| `SportsRetrievalOutput` | SportsRetrievalOutput.cs | retrieve stage output; owns grounding + availability diagnostics |
| `SharpPublicContext` | SportAnalysisContracts.cs | sharp vs public betting split from actionnetwork |
| `SharpPublicLookupResult` | ActionNetworkClient.cs | wrapper around SharpPublicContext? with status + MissingReason |
| `SignalAvailabilityRecord` | AgentRunContracts.cs | per-signal availability + quality record; stored in OutputJson |
| `SignalQualityEvaluator` | SignalQualityEvaluator.cs | deterministic signal quality rules; enriches availability records with quality, decision use, confidence effect, and follow-up signals |
| `SportsCognitivePhases` | SportAnalysisContracts.cs / sports.py | internal 4-phase cognitive artifact stored in OutputJson |
| `EvaluatorOutput` | EvaluatorOutput.cs | calibrated confidence from evaluate stage |
| `AgentRunExecutionResult` | AgentRunContracts.cs | final artifact; serialized to OutputJson |
| `AgentRunEvaluationDto` | AgentRunContracts.cs | response shape for GET /evaluation read endpoint |

---

## pipeline stages

### stage 1: retrieve
**class:** `SportsRetriever` (implements `ISportsRetriever`)
**signature:** `Task RetrieveAsync(SportsRunArtifact artifact, CancellationToken ct)`
**file:** `platform/dotnet/DevCore.Api/AgentRuns/SportsRetriever.cs`

Reads `artifact.Input` and `artifact.GameDate`.
Retrieves grounded evidence from external APIs per competition code.
Calls `artifact.RecordRetrieve(output, status, note?)`.

| competition | data source | client |
|---|---|---|
| nfl, ncaaf | current spread | OddsMarketClient |
| nfl, ncaaf | sharp vs public split | ActionNetworkClient |
| mlb | probable starters | MlbStarterClient |
| nba, ncaamb | rest/schedule + spread | EspnBasketballScheduleClient + OddsMarketClient |
| nba, ncaamb | sharp vs public split | ActionNetworkClient |

**step status logic:**
- `MaxGroundedSignals == 0` or all sources returned data → `Succeeded`
- some sources returned null → `Degraded` with note ("N of M signals grounded")
- all sources returned null → `Degraded` with note ("no signals grounded -- priors-only run")

**maxgroundedsignals per competition (current):**
| competition | max | sources |
|---|---|---|
| nfl | 2 | spread (odds api) + sharp/public (actionnetwork) |
| ncaaf | 2 | spread (odds api) + sharp/public (actionnetwork) |
| nba | 3 | schedule (espn) + spread (odds api) + sharp/public (actionnetwork) |
| ncaamb | 3 | schedule (espn) + spread (odds api) + sharp/public (actionnetwork) |
| mlb | 1 | probable starters (statsapi.mlb.com) |

A degraded retrieve does not stop the run. The artifact's `Publishability` is downgraded to
`PublishableWithCaveats`. The analyzer receives explicit no-data instructions and falls back gracefully.
The evaluator dampens confidence according to grounded signal count.
ActionNetworkClient uses the first odds entry with all four percentage fields populated.
If the provider returns the same matchup with home and away flipped, the client preserves the requested
home/away orientation and swaps the percentages to match.

**Signal Availability Diagnostics v1 (2026-05-09) implementation:**
`ActionNetworkClient.GetSharpPublicDataAsync` now returns `SharpPublicLookupResult` instead of `SharpPublicContext?`. Every failure path carries a `Status` (`missing` or `not_attempted`) and a `MissingReason` string distinguishing: `no_matching_game`, `provider_returned_null_pct`, `provider_returned_empty_odds`, `unsupported_competition`, `provider_error`, `invalid_date`. `SportsRetrievalOutput` computes `SignalAvailability[]` from all signal retrieval outcomes — grounded signals use `status=grounded`; missing/not-attempted carry the detailed reason. `AgentRunExecutionResult` stores `SignalAvailability` in OutputJson. `GET /api/agent-runs/{id}/artifact` exposes it. The `/dev/artifacts` Angular page renders a Signal Availability table. The calibration report PS1 includes a Signal Availability section per matchup.

**Sharp/Public Fallback Ladder v1 (2026-05-11) implementation:**
`SignalFollowUpRecord` is extended with four additive optional fields: `FallbackType`, `Equivalence`, `ConfidencePermission`, `PosturePermission`. `SignalFollowUpEvaluator` classifies every follow-up against a six-tier ladder: `exact_recovery` / `source_substitution` / `fidelity_downgrade` / `adjacent_proxy` / `lateral_proxy` / `unavailable_with_reason`. Two invariants are pinned: lateral proxies never lift confidence on their own, and only `exact_recovery` or `source_substitution` may permit `aggressive_allowed_if_corroborated`. `line_movement` is classified as `adjacent_proxy` / `adjacent` regardless of implementation status — it answers "did the price move" not "what is the public/sharp split", so it is not an equivalent substitute for missing `sharp_public`. Ladder fields are observational on the artifact in this slice; `SportsEvaluator` calibration was not modified. The Claude Code skill `dai-signal-follow-up-diagnostics` was updated to honor the ladder. Line Movement Proxy v1 is now explicitly deferred — the platform should walk the ladder top-down before reaching for adjacent proxies. See `02 Platform/architecture/cognitive-factory/signal-fallback-ladder.md`.

**Signal Follow-Up Diagnostics v1 (2026-05-10) implementation:**
`SignalFollowUpEvaluator` is a new deterministic static class in `DevCore.Api/AgentRuns/`. It is called from `SportsRetrievalOutput`'s constructor immediately after `SignalQualityEvaluator.Enrich`. It walks every `SignalAvailabilityRecord.FollowUpSignals` entry and resolves it into a `SignalFollowUpRecord` carrying `TriggeredBy`, `Signal`, `Status` (`grounded` / `missing` / `unavailable` / `not_implemented` / `candidate`), `Source`, `Reason` (`already_grounded` / `future_signal_candidate` / `provider_unavailable` / `primary_signal_missing` / `proxy_candidate` / `not_required`), `DecisionUse` (`proxy_candidate` / `confirmation_candidate` / `already_available` / `missing_confirmation` / `not_usable` / `directional_context`), and a `Detail` string. Follow-up signals are decision clues, not flat checkboxes — a follow-up signal can be grounded, missing, unavailable, not implemented, or only a candidate, and this evaluator makes the difference inspectable. `line_movement` is currently a follow-up candidate that is `not_implemented` whenever `sharp_public` is missing (we have no historical line snapshot anywhere) and a `candidate` when `sharp_public` is grounded; this prepares for a future `Line Movement Proxy v1` slice but adds no backup provider and no outcome reconciliation in this slice. `SignalFollowUps` is added to `AgentRunExecutionResult` and `AgentRunArtifactDto`. `GET /api/agent-runs/{id}/artifact` exposes it. `/dev/artifacts` adds a "Signal Follow-Up Diagnostics" section above Signal Availability. The calibration report renders a `## Signal Follow-Up Diagnostics` table per matchup; the recommendation basis now says "follow-up path exists but line_movement is not implemented yet" when applicable. Per-signal rules live in private methods inside `SignalFollowUpEvaluator` so a future slice can refine `sharp_public` or `market` independently. There is no rules engine and no scoring math.

**Signal Quality Model v1 (2026-05-09) implementation:**
`SignalQualityEvaluator` is a new deterministic static class in `DevCore.Api/AgentRuns/`. It is called from within `SportsRetrievalOutput`'s constructor after all availability records are built. It enriches each `SignalAvailabilityRecord` with four quality fields: `Quality` (strong / usable / unavailable), `DecisionUse` (the role the signal plays in the decision), `FollowUpSignals` (signals to check next when this one is unavailable or needs confirmation), and `ConfidenceEffect` (the effect this signal's state has on posture aggressiveness: support / support_cautiously / dampen / block_aggressive_posture / neutral). Market quality depends on whether `sharp_public` is also grounded — `block_aggressive_posture` is assigned to `sharp_public` when missing, signaling that aggressive read postures are not warranted without confirmation. `SignalAvailabilityRecord` in `AgentRunContracts.cs` is extended with these four nullable fields. `SignalAvailabilityDto` in `agent-run.model.ts` carries them. The `/dev/artifacts` Angular table expands to 8 columns. The calibration report PS1 shows quality and confidence effect per signal and flags `signal_quality_blocks_aggressive_posture` in per-run assessment when any signal carries that effect.

**Signal Source Review v1 + Signal Availability Diagnostics v1 (2026-05-08) findings:**
Three schema bugs in `ActionNetworkClient` were identified and corrected:

1. **Team names**: the real API uses `away_team_id`/`home_team_id` integers + a `teams[]` array with `full_name`. The client previously looked for `away_team`/`home_team` nested sub-objects. Team lookup now falls back to `TryGetTeamNamesFromArray` using the `teams` array.

2. **Odds field name**: without the `bookIds` query parameter, `odds: []` always. Adding `?bookIds=15` causes the odds array to be populated with entries that include the percentage fields. The client URL now includes `&bookIds=15`.

3. **Percentage field names**: the real field names are `spread_home_public`, `spread_away_public`, `spread_home_money`, `spread_away_money` — not `home_spread_pct`, `away_spread_pct`, `home_money_pct`, `away_money_pct` as originally assumed. The client now parses the correct field names.

**Data availability by context**: percentage splits are populated for regular season NFL and NBA games. During NBA and NFL playoffs, these fields return null across all checked book IDs and dates. This is a provider limitation of the current endpoint, not a parsing issue. `sharp_public` will degrade gracefully (null → missing signal) during playoffs.

`sharp_public` expected signals are correct and should remain as-is. The signal degrades gracefully (partial retrieval) without blocking the run. Confidence is dampened by the evaluator when grounded signal count is below max. During regular season play, `sharp_public` should now be grounded when betting volume is sufficient.

---

### stage 2: analyze
**class:** `SportsAnalyzer` (implements `ISportsAnalyzer`)
**signature:** `Task AnalyzeAsync(SportsRunArtifact artifact, CancellationToken ct)`
**file:** `platform/dotnet/DevCore.Api/AgentRuns/SportsAnalyzer.cs`

Reads `artifact.Input`, `artifact.GameDate`, `artifact.RetrievalOutput`.
Builds `SportsAnalysisRequest` and calls `FastApiClient.AnalyzeSportsMatchupAsync`.
FastAPI dispatches to `sports_analyzer.py` → gpt-4o-mini with sport-specific system prompt.
Calls `artifact.RecordAnalyze(response, Succeeded)`.

This is the only AI step. One model call per run. json_object mode, temperature 0.3.
Confidence from the response is provisional -- the evaluator calibrates it in the next stage.
The analyze response now includes `lean_side` ("home", "away", or null) alongside the narrative `lean` string.
The prompt instructs the model to emit `lean_side` consistent with its lean text.
FastAPI normalizes: only "home" and "away" are accepted -- any other value is clamped to null.
`lean_side` flows through the artifact into `AgentRunExecutionResult` and is denormalized to `AgentRun.LeanSide`.
The analyze response now also carries `signals_used`: a compact list of signal category keys the model self-reported incorporating (e.g. `["market", "situational"]`). This is the model-side complement to `GroundedSignals` from the retrieve layer. Together they form the retrieved vs. used picture needed for calibration.
The analyze response now also carries cognitive artifact v1:
- `phases`: internal 4-phase object with perceive, interrogate, discern, decide
- `posture`: validated read stance, one of play/pass/monitor/wait/compare/avoid or null
- `counter_case`: strongest case against the lean
- `watch_for`: fragile assumption or condition to monitor
- `what_would_change_the_read`: short conditions that would flip or modify the read

FastAPI validates posture and clamps unknown values to null. Missing phase fields are safe and do not fail parsing.
Top-level `counter_case` and `watch_for` fall back to `phases.interrogate.balance` and
`phases.interrogate.stress` when the deliver extracts are absent.

Dev Calibration Report Assessment v1 (2026-05-08) adds deterministic automated assessment to calibration reports generated by `run-artifact-calibration.ps1`. It flags missing phase actions, generic output, possibly unsupported claims (injuries/resilience/trends mentioned without grounded signals), confidence/evidence mismatch, missing signals, and candidate next slices. Assessment happens before outcome reconciliation so artifact quality can be improved independently of game results. No model calls are made during assessment.

Cognitive prompt tightening v1.5 (2026-05-08) added the following prompt-level constraints to `_JSON_SHAPE` in `sports_analyzer.py`:
- `frame` must name the rest situation explicitly: rest differential when unequal, equal rest when teams share the same rest; must integrate context from detect rather than repeat a bare matchup slug
- `balance` must cite a specific signal or name the missing-signal limitation explicitly (e.g. "sharp/public data is missing, so this is a market-only lean"); prohibited phrases expanded: "could outperform", "has potential", "may surprise", "talent", "strong team", "recent performance"; must not invent a plausible team trait
- `stress` must prefer concrete fragilities: missing sharp_public confirmation, large spread cover risk, market-only lean, equal rest limiting schedule edge, missing injury or lineup data
- `reframe` must not mention hidden strengths, resilience, performance trends, recent form, travel, weather, injuries, or player availability unless present in the supplied context
- `test` must state whether a market-only read stands on the spread alone or needs independent confirmation when no other signals are present
- `filter` must name what is admitted, what is missing, and what should be discounted; if a signal category is absent, name it as missing
- `counter_case` must be grounded in one or more of: missing sharp_public, spread size or cover risk, rest parity, lack of supporting signals, or market-only limitation
- `watch_for` should name a spread move as the watch condition when sharp_public is missing
- anti-fabrication header expanded to also prohibit: hidden strengths, matchup history, performance trends; added rule to name the limitation instead of inventing a team trait when no grounded counterpoint exists

Cognitive prompt tightening v1 (2026-05-07) added the following prompt-level constraints to `_JSON_SHAPE` in `sports_analyzer.py`:
- `frame` must include the spread when market data is present and name the rest differential when schedule data is present
- `balance` must cite the specific signal being pushed back against; if counter-evidence is thin, say so rather than inventing a counter
- `reframe` and `test` must derive from available signals; fabrication is prohibited
- `listen` is required (non-null) whenever any market or sharp/public block was provided; null is only valid when zero external blocks were present
- `counter_case` must cite the specific signal or condition; generic phrases ("has the potential", "could outperform", "may surprise", "talent") are prohibited
- `watch_for` must name something specific and observable; vague phrases ("inconsistent performance", "competitive edge", "performance trends") are prohibited
- `what_would_change_the_read` items must each name a specific condition, event, or data point; 1-2 strong items preferred over 3 generic ones
- anti-fabrication rule explicitly prohibits claiming resilience, recent form, injuries, travel, player availability, or team trends unless they appear in the supplied context blocks
If this stage throws (model error, FastAPI down), AgentRunService catches the exception
(except OperationCanceledException), calls `artifact.RecordAnalyzeFailed(ex.Message)` and
`composer.ComposeFailedRun(artifact)`, then re-throws as `AnalysisPipelineException`.
The controller catches `AnalysisPipelineException` specifically and persists the failure artifact
as OutputJson before marking the run "failed" and re-throwing.

---

### stage 3: evaluate
**class:** `SportsEvaluator` (implements `ISportsEvaluator`)
**signature:** `void Evaluate(SportsRunArtifact artifact)`
**file:** `platform/dotnet/DevCore.Api/AgentRuns/SportsEvaluator.cs`

Reads `artifact.AnalyzerOutput`, `artifact.RetrievalOutput`, `artifact.Input.Competition`.
Pure computation -- no I/O, no model calls. Always records `Succeeded`.
Calls `artifact.RecordEvaluate(result, Succeeded)`.

Calibration tiers by grounded signal count:

| tier | condition | dampening | clamp range |
|---|---|---|---|
| priors-only | 0 grounded | 0.75 | [0.30, 0.60] |
| partial | some but not all | 0.90 | [0.35, 0.75] |
| fully grounded | all available | 1.00 | [0.35, 0.85] |

`AggregateConfidence` is the final confidence stored in the decision artifact.
`AnalyzerConfidence` is stored alongside for learning loop comparison.
`ConfidenceBand` labels the calibrated confidence as "high", "medium", or "low".

---

### stage 4: compose
**class:** `SportsComposer` (implements `ISportsComposer`)
**signature:** `void Compose(SportsRunArtifact artifact)`
**file:** `platform/dotnet/DevCore.Api/AgentRuns/SportsComposer.cs`

Reads all prior stage outputs from the artifact.
Assembles `AgentRunExecutionResult` including all pipeline metadata.
Calls `artifact.RecordCompose(result, Succeeded)`.
`artifact.FinalResult` being non-null signals the pipeline completed successfully.
`SportsComposer` passes the internal `CognitivePhases` object into OutputJson and maps only the compact
deliver-layer fields to the final response DTO. `EvidenceRichness` is set from
`retrieval.GroundedSignals.Length`, not from the model. `SignalAvailability` is now passed from
`retrieval.SignalAvailability` into `AgentRunExecutionResult` and persisted in OutputJson.
It is exposed via `GET /api/agent-runs/{id}/artifact` and rendered on the `/dev/artifacts` page.
It is not surfaced in `AgentRunResultDto` or the normal user-facing sports card.

---

## orchestrator

**class:** `AgentRunService` (implements `IAgentRunService`)
**file:** `platform/dotnet/DevCore.Api/AgentRuns/AgentRunService.cs`

Creates `SportsRunArtifact`, calls the four stages in order, returns `artifact.FinalResult`.
Contains no business logic -- coordination only.

```
var artifact = new SportsRunArtifact(req.Input, gameDate, agentRunId);
await retriever.RetrieveAsync(artifact, ct);  // external apis
await analyzer.AnalyzeAsync(artifact, ct);    // one model call
evaluator.Evaluate(artifact);                 // deterministic calibration
composer.Compose(artifact);                   // final assembly
return artifact.FinalResult;
```

---

## request entry point

**class:** `AgentRunsController`
**file:** `platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs`

Receives `CreateAgentRunRequest`, persists pending row (including `Competition` and `GameDate`), calls `IAgentRunService.ExecuteAsync`,
updates the row, returns `AgentRunResultDto`. No knowledge of the artifact or pipeline stages.

---

## full request path

```
Angular AnalyzerComponent
  -> POST /api/agent-runs (CreateAgentRunRequest)
  -> AgentRunsController.Create
    -> save pending AgentRun row
    -> AgentRunService.RunSportsMatchupPipelineAsync
        artifact = new SportsRunArtifact(input, gameDate, agentRunId)
        1. ISportsRetriever.RetrieveAsync(artifact)   [external http; records Succeeded or Degraded]
        2. ISportsAnalyzer.AnalyzeAsync(artifact)    [one gpt-4o-mini call; records Succeeded]
        3. ISportsEvaluator.Evaluate(artifact)       [pure computation; records Succeeded]
        4. ISportsComposer.Compose(artifact)         [pure mapping; records Succeeded; sets FinalResult]
    -> return artifact.FinalResult (AgentRunExecutionResult)
    -> update AgentRun row (completed, OutputJson includes pipeline metadata)
  -> return AgentRunResultDto (lean, summary, confidence, factors, read stance fields)
```

## internal artifact review surface

Dev Artifact Review Page v1 adds `/dev/artifacts` in `apps/sports-app`.
It lets a builder paste a completed AgentRun ID and inspect the curated internal artifact through `GET /api/agent-runs/{agentRunId}/artifact`.
The page is for builder learning, debugging, and quality review.
It is not linked from the main nav and is not part of the main user-facing sports read.
explicit admin/dev role gating is deferred until the app has a role model.

Dev Artifact Review Selector v1 adds a recent-runs selector to `/dev/artifacts`.
`GET /api/agent-runs/recent` returns the most recent 25 sports matchup analysis runs scoped to the calling tenant and user.
The selector lets a builder pick a run without manually querying the database for AgentRun IDs.
It uses only denormalized columns and InputJson for display; no OutputJson parsing in the list response.

Dev Upcoming Artifact Runs v1 adds a "Run Upcoming Samples" section to `/dev/artifacts`.
`GET /api/competitions/{code}/upcoming?days=7` returns all upcoming scheduled games for a competition (no team-pair filtering) using the Odds API.
The section lets a builder select up to 10 upcoming games (checkboxes) and batch-run them as AgentRuns using the existing sports pipeline.
Execution is sequential: one `POST /api/agent-runs` per game, awaited in turn. No background jobs, no cron, no billing changes.
Per-game progress is shown inline with status badges. Completed runs show a truncated agentRunId for quick cross-reference.
After the batch completes the recent-runs selector above refreshes automatically so runs are immediately inspectable.
`OddsScheduleClient.GetAllUpcomingEventsAsync` is the new schedule method; it reuses the Odds API infrastructure without team-pair filtering.
Backend: days default 7, max 14; result capped at 50 events. Degrades gracefully (empty array) when API key is missing.
Frontend: hard max 10 selected games; Run button disabled until at least one game is checked.
6 integration tests added to `SportsReferenceControllerTests` covering 404 paths, graceful degradation, and days param handling.

Dev Calibration Polish v1 fixes three real-usage friction points in the builder workflow:

1. **Signal alias normalization in `SportsQualityChecker` rule 4.** The FastAPI analyzer prompt instructs the model to use a different vocabulary than the platform's canonical grounded signal names (e.g. model emits `rest_fatigue`; platform grounds `rest_schedule`). Without normalization, rule 4 fired a false quality warning. `SportsQualityChecker` now maps model-emitted aliases to platform canonical names before checking grounded-set membership. Canonical platform signals remain: `market`, `sharp_public`, `rest_schedule`, `starting_pitching`. Conservative aliases include `rest_fatigue → rest_schedule`, `public_sharp → sharp_public`, `starter → starting_pitching`. The warning message uses the canonical name. 4 new tests added to `SportsQualityCheckerTests` covering the alias paths.

2. **Cognitive phase completeness on `/dev/artifacts`.** Previously, phase fields with null or empty output were silently hidden, making it impossible to tell whether a field was missing from the model response or simply not displayed. The Dev Artifact Review page now always renders all 12 expected phase actions (Perceive: Detect/Frame/Aim, Interrogate: Balance/Stress/Reframe, Discern: Test/Listen/Filter, Decide: Calibrate/Posture/Voice) with explicit "Not recorded" for absent fields. Builders can now distinguish missing model output from UI omission.

3. **Actionable empty and error states in the upcoming games section.** The "no games found" empty state and the error state now carry actionable guidance text instead of bare status messages.

Dev Artifact Calibration Harness v1 adds `scripts/dev/sports/run-artifact-calibration.ps1`.
It lets a builder run a small capped batch of upcoming sports analyses from the command line, fetch the resulting artifacts, and write a readable calibration report into `dai-vault/04 Products/sports-v1/calibration/`.
It is a manual feedback loop for builder use only — no scheduled jobs, no background automation, no billing changes.
Parameters: `-Competition`, `-Take` (cap: 10), `-Days` (cap: 14), `-ApiBaseUrl`, `-BearerToken`, `-DryRun`, `-Force`.
Auth: reads bearer token from `-BearerToken` param or `DAI_DEV_BEARER_TOKEN` env var; falls back to local dev bypass (`Dev:EnableBypassAuth`).
Report format: metadata, summary table, quality warnings grouped by type, cognitive phase completeness table, blank manual review notes section.
Raw artifact JSON files are written alongside the report in `calibration/artifacts/`.
See `scripts/dev/sports/README.md` for usage examples.

---

## what is stored in OutputJson (new vs before)

| field | before | after |
|---|---|---|
| Lean | yes | yes |
| Summary | yes | yes |
| Confidence | yes | yes |
| Factors | yes | yes |
| GroundedSignals | yes | yes |
| AnalyzerConfidence | yes | yes |
| CognitivePhases | no | yes -- internal 4-phase artifact in OutputJson only |
| Posture | no | yes -- compact read stance, surfaced as Read Stance |
| CounterCase | no | yes -- compact counter-case field surfaced to the UI |
| WatchFor | no | yes -- compact watch condition surfaced to the UI |
| WhatWouldChangeTheRead | no | yes -- compact list surfaced to the UI when populated |
| EvidenceRichness | no | yes -- nullable grounded signal count from retrieval |
| Publishability | no | yes -- enum name string |
| DegradationNotes | no | yes -- string[] of any degradation messages |
| PipelineSteps | no | yes -- [{StepName, Status, Note?}] for all 4 stages |

API response shape to the UI remains compact. It does not expose the nested `CognitivePhases` object.

---

## publishability rules (v1)

| situation | publishability | why |
|---|---|---|
| all sources returned data | Publishable | clean grounded run |
| some sources returned null | PublishableWithCaveats | partial grounding; still valid with dampened confidence |
| all sources returned null (maxGrounded > 0) | PublishableWithCaveats | priors-only run; valid with heavy dampening |
| no sources configured (maxGrounded == 0) | Publishable | priors-only is expected for this competition tier |
| analyze throws | NotPublishable | caught in AgentRunService; failure artifact persisted as OutputJson |

---

## di registration

```csharp
// scoped: depend on typed http clients (transient via factory)
builder.Services.AddScoped<ISportsRetriever, SportsRetriever>();
builder.Services.AddScoped<ISportsAnalyzer, SportsAnalyzer>();

// singleton: pure computation with no i/o or mutable state
builder.Services.AddSingleton<ISportsEvaluator, SportsEvaluator>();
builder.Services.AddSingleton<ISportsComposer, SportsComposer>();

builder.Services.AddScoped<IAgentRunService, AgentRunService>();
```

`SportsRunArtifact` is not registered -- it is created inline in `RunSportsMatchupPipelineAsync`.

---

## what is and is not AI

| stage | AI? | token cost |
|---|---|---|
| retrieve | no -- external http | none |
| analyze | yes -- gpt-4o-mini | ~$0.01-0.03 per run |
| evaluate | no -- static rules | none |
| compose | no -- field mapping | none |

---

## automated tests (as of 2026-05-07)

test project: `platform/dotnet/DevCore.Api.Tests`
framework: xUnit 2.9.x, real instances for pure logic, hand-written fakes at the two i/o boundaries.

| file | tests | what is locked down |
|---|---|---|
| SportsRunArtifactTests | 7 | RecordRetrieve publishability + notes; RecordAnalyzeFailed step/publishability/notes |
| SportsEvaluatorTests | 7 | calibration tiers (priors-only, partial, fully grounded); confidence band labels; step recording |
| SportsComposerTests | 11 | Compose confidence ownership; pipeline step list; nullable evidence richness compatibility; ComposeFailedRun lean/confidence/publishability/steps |
| AgentRunServiceTests | 7 | analyze failure wrapping; AnalysisPipelineException artifact; cancellation propagation; success path with and without sharp_public |
| RunEvaluatorTests | 13 | WinningSide mapping; correct/incorrect/inconclusive paths for all outcome types |
| AgentRunsControllerTests | 9 | OutputJson persistence on analyze failure; ErrorMessage from InnerException; RecordOutcome 201/409/404; evaluation persisted on outcome; GetEvaluation 200/404-no-run/404-no-eval |
| SportsQualityCheckerTests | 16 | 5 deterministic quality rules; signal alias normalization (rest_fatigue, public_sharp, starter); unknown signals still warn; clean-pass case |
| SportsReferenceControllerTests | 6 | 404 for unknown code; 404 for unseeded competition; 200+empty when API key absent; days=0 defaults; days=999 clamps; response is plain array |
| SportsRetrieverTests | 2 | sharp_public included in grounded signals when available; graceful degraded retrieve when sharp_public is missing |
| SportsAnalyzerTests | 1 | .NET analyze request forwards compact sharp_public context into the single FastAPI model call |
| ActionNetworkClientTests | 12 | correct parse; case-insensitive match; partial name match; flipped home/away mapping; later usable book fallback; empty games; no match; missing books_odds; empty books_odds; null percentages; unsupported competition; non-200 response |
| tests/test_sports_analyzer.py (python) | 67 | lean_side parsing; signals_used validation; sharp_public prompt context; cognitive phases parsing; posture validation; deliver extract fallbacks |

run command: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj`

total: 125 .NET tests, 67 Python tests, all passing in the current validation pass.

---

## migration notes

`AddAgentRunOutcomeColumns` (20260424130353) contained duplicate InsertData for Teams 101-330 (NFL/NBA/MLB).
those teams were already seeded by `AddSportsReferenceData` which was applied earlier.
the duplicate inserts caused a PRIMARY KEY violation when `dotnet ef database update` was run.

fix applied 2026-04-25: removed the InsertData block and corresponding DeleteData blocks from
`AddAgentRunOutcomeColumns.cs` Up()/Down(). the migration was not yet applied when it failed
(ef rolls back on error), so the edit was safe.

because the running API locked DevCore.Api's bin DLL, migrations were applied directly via sqlcmd
using a hand-written SQL script (C:\temp\apply_migrations.sql) covering all four pending migrations
in order: MakeCompetitionFirstClass, AddAgentRunOutcomeColumns, AddAgentRunOutcome, AddAgentRunEvaluation.

**future note**: after pulling migration files, always run `dotnet ef database update --project DevCore.Data
--startup-project DevCore.Api` with the API server stopped. if the API is running and the migration has
duplicate seed data, the rollback will hide the schema changes -- stop the server first.

---

## known gaps (as of 2026-04-25)

- sharp/public split is now live for nfl, ncaaf, nba, ncaamb via ActionNetworkClient.
  actionnetwork uses an unofficial undocumented api -- schema is not under our control.
  client returns null gracefully if the schema changes; run degrades to priors-only for this signal.
  mlb sharp/public is deferred -- spread betting split is not the primary quality driver for baseball.
- confidence calibration parameters are undocumented interim estimates pending the learning loop.
- Competition and GameDate are first-class columns on AgentRun (added in migration AddAgentRunOutcomeColumns).
- AgentRunOutcome entity is live. Raw game results are recorded via POST /api/agent-runs/{id}/outcome.
- AgentRunEvaluation entity is live. Derived evaluation (correct/incorrect/inconclusive) is computed and persisted atomically at outcome recording time.
- GET /api/agent-runs/{id}/evaluation is live. Returns AgentRunEvaluationDto (200) or 404 when the run does not exist for the tenant or no evaluation has been computed yet.
- FastAPI now returns `lean_side` alongside the narrative `lean`. The evaluation loop is active — runs produce correct/incorrect instead of defaulting to inconclusive.
- Confidence calibration analysis (was the confidence accurate relative to the outcome?) is deferred — the data is stored but no comparison logic exists yet.
