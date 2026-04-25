# current sports analysis flow

Last updated: 2026-04-25 (lean_side slice: FastAPI returns structured lean_side; evaluation loop is now active)
Reflects code state after the lean_side activation slice.

---

## overview

A sports.matchup.analysis run executes four pipeline stages against a single shared working artifact.
Each stage reads its required inputs from the artifact and records its output and status back.
AgentRunService is the thin entry orchestrator -- it creates the artifact and calls the four stages in order.
There is exactly one model call per run, in the analyze stage.

The artifact carries publishability, degradation notes, and per-stage step results.
These are persisted in OutputJson for observability and the future learning loop.
They are not surfaced to the UI -- the API response shape is unchanged.

---

## key types

| type | file | role |
|---|---|---|
| `SportsRunArtifact` | SportsRunArtifact.cs | working artifact; flows through all stages |
| `PipelineStepResult` | PipelineModels.cs | recorded outcome of one stage |
| `PipelineStepStatus` | PipelineModels.cs | Succeeded / Degraded / Skipped / Failed |
| `PublishabilityStatus` | PipelineModels.cs | Publishable / PublishableWithCaveats / NotPublishable |
| `SportsRetrievalOutput` | SportsCollectorOutput.cs | retrieve stage output; owned signal grounding |
| `EvaluatorOutput` | EvaluatorOutput.cs | calibrated confidence from evaluate stage |
| `AgentRunExecutionResult` | AgentRunContracts.cs | final artifact; serialized to OutputJson |

---

## pipeline stages

### stage 1: retrieve
**class:** `SportsRetriever` (implements `ISportsRetriever`)
**signature:** `Task RetrieveAsync(SportsRunArtifact artifact, CancellationToken ct)`
**file:** `platform/dotnet/DevCore.Api/AgentRuns/SportsCollector.cs`

Reads `artifact.Input` and `artifact.GameDate`.
Retrieves grounded evidence from external APIs per competition code.
Calls `artifact.RecordRetrieve(output, status, note?)`.

| competition | data source | client |
|---|---|---|
| nfl, ncaaf | current spread | OddsMarketClient |
| mlb | probable starters | MlbStarterClient |
| nba, ncaamb | rest/schedule + spread | EspnBasketballScheduleClient + OddsMarketClient |

**step status logic:**
- `MaxGroundedSignals == 0` or all sources returned data → `Succeeded`
- some sources returned null → `Degraded` with note ("N of M signals grounded")
- all sources returned null → `Degraded` with note ("no signals grounded -- priors-only run")

A degraded retrieve does not stop the run. The artifact's `Publishability` is downgraded to
`PublishableWithCaveats`. The analyzer receives explicit no-data instructions and falls back gracefully.
The evaluator dampens confidence according to grounded signal count.

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
  -> return AgentRunResultDto (lean, summary, confidence, factors -- same as before)
```

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
| Publishability | no | yes -- enum name string |
| DegradationNotes | no | yes -- string[] of any degradation messages |
| PipelineSteps | no | yes -- [{StepName, Status, Note?}] for all 4 stages |

API response shape to the UI is unchanged (uses AgentRunResultDto, not AgentRunExecutionResult).

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

## automated tests (as of 2026-04-23)

test project: `platform/dotnet/DevCore.Api.Tests`
framework: xUnit 2.9.x, real instances for pure logic, hand-written fakes at the two i/o boundaries.

| file | tests | what is locked down |
|---|---|---|
| SportsRunArtifactTests | 7 | RecordRetrieve publishability + notes; RecordAnalyzeFailed step/publishability/notes |
| SportsEvaluatorTests | 7 | calibration tiers (priors-only, partial, fully grounded); confidence band labels; step recording |
| SportsComposerTests | 10 | Compose confidence ownership; pipeline step list; ComposeFailedRun lean/confidence/publishability/steps |
| AgentRunServiceTests | 6 | analyze failure wrapping; AnalysisPipelineException artifact; cancellation propagation; success path |
| RunEvaluatorTests | 13 | WinningSide mapping; correct/incorrect/inconclusive paths for all outcome types |
| AgentRunsControllerTests | 6 | OutputJson persistence on analyze failure; ErrorMessage from InnerException; RecordOutcome 201/409/404; evaluation persisted on outcome |
| tests/test_sports_analyzer.py (python) | 13 | lean_side parsing: valid values, absent field, invalid clamped to null; required field validation |

run command: `dotnet test DevCore.Api.Tests/DevCore.Api.Tests.csproj`

total: 56 tests, all passing.

---

## known gaps (as of 2026-04-24)

- sharp/public split is in v1-scope.md but has no retriever implementation.
  actionnetwork client does not exist. adding it only requires a new client, a new context type,
  a new SportsRetrievalOutput field, and a new branch in SportsRetriever.RetrieveAsync.
- confidence calibration parameters are undocumented interim estimates pending the learning loop.
- Competition and GameDate are first-class columns on AgentRun (added in migration AddAgentRunOutcomeColumns).
- AgentRunOutcome entity is live. Raw game results are recorded via POST /api/agent-runs/{id}/outcome.
- AgentRunEvaluation entity is live. Derived evaluation (correct/incorrect/inconclusive) is computed and persisted atomically at outcome recording time.
- FastAPI now returns `lean_side` alongside the narrative `lean`. The evaluation loop is active — runs produce correct/incorrect instead of defaulting to inconclusive.
- Confidence calibration analysis (was the confidence accurate relative to the outcome?) is deferred — the data is stored but no comparison logic exists yet.
