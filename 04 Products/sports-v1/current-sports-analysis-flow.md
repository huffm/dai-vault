# current sports analysis flow

Last updated: 2026-04-30 (cognitive artifact v1: 4-phase read stance slice)
Reflects code state after compact cognitive artifact integration.

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
| `SportsRetrievalOutput` | SportsRetrievalOutput.cs | retrieve stage output; owned signal grounding |
| `SharpPublicContext` | SportAnalysisContracts.cs | sharp vs public betting split from actionnetwork |
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
ActionNetworkClient uses the first `books_odds` entry with all four percentage fields populated.
If the provider returns the same matchup with home and away flipped, the client preserves the requested
home/away orientation and swaps the percentages to match.

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
`retrieval.GroundedSignals.Length`, not from the model.

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

## automated tests (as of 2026-04-25)

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
| SportsRetrieverTests | 2 | sharp_public included in grounded signals when available; graceful degraded retrieve when sharp_public is missing |
| SportsAnalyzerTests | 1 | .NET analyze request forwards compact sharp_public context into the single FastAPI model call |
| ActionNetworkClientTests | 12 | correct parse; case-insensitive match; partial name match; flipped home/away mapping; later usable book fallback; empty games; no match; missing books_odds; empty books_odds; null percentages; unsupported competition; non-200 response |
| tests/test_sports_analyzer.py (python) | 67 | lean_side parsing; signals_used validation; sharp_public prompt context; cognitive phases parsing; posture validation; deliver extract fallbacks |

run command: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj`

total: 85 .NET tests, 67 Python tests, all passing in the current validation pass.

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
