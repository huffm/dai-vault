# current agent run contract

**date:** 2026-05-01 (run artifact inspection v1)
**derived from:** actual code in `platform/dotnet/` and `apps/sports-app/`
**status:** reflects what is implemented today — not a design target

---

## database entity: AgentRun

file: `DevCore.Domain/Agentic/AgentRun.cs`
table: `AgentRuns` (via EF Core, `DevCore.Data/AppDbContext.cs`)
migration added: `20260311223224_AddAgentRuns`

| column | type | notes |
|---|---|---|
| AgentRunKey | bigint | internal PK, identity |
| AgentRunId | guid | public-facing identifier, sequential |
| TenantKey | bigint | FK to Tenants, NoAction on delete |
| RequestedByUserKey | bigint | FK to Users, NoAction on delete |
| AgentProfileKey | bigint? | FK to AgentProfiles, nullable — not enforced at runtime |
| RunType | string | free-form string; currently `"sports.matchup.analysis"` |
| Status | string | check constraint: `pending \| completed \| failed` |
| InputJson | string | raw JSON of the request input — no schema enforced |
| OutputJson | string | raw JSON of the AI service response — no schema enforced |
| ErrorMessage | string? | populated on failure |
| CorrelationId | string? | set by `AgentRunsController` to `HttpContext.TraceIdentifier` (ASP.NET short request trace id) |
| StartedUtc | datetime | set when row is created |
| CompletedUtc | datetime? | set after ExecuteAsync returns |
| DurationMs | long? | computed from StartedUtc/CompletedUtc |
| Competition | string? | denormalized from input (e.g. `"nfl"`); indexed; populated at row creation |
| GameDate | date? | denormalized from input; indexed; populated at row creation so outcomes can join runs |
| LeanSide | string?(8) | structured lean direction: `"home"`, `"away"`, or null; denormalized from `AgentRunExecutionResult.LeanSide` at run completion; null until fastapi returns `lean_side` |

### what OutputJson currently contains

OutputJson is written as the raw serialized `AgentRunExecutionResult`:

```json
{
  "Lean": "Edge toward Chiefs based on rest advantage and line movement since open.",
  "Summary": "...",
  "Confidence": 0.64,
  "GroundedSignals": ["market"],
  "MissingSignals": ["sharp_public"],
  "AnalyzerConfidence": 0.76,
  "Factors": ["Market: Current spread supports the lean.", "Rest: Chiefs have the schedule edge."],
  "CognitivePhases": {
    "Perceive": { "Detect": ["..."], "Frame": "...", "Aim": ["..."] },
    "Interrogate": { "Balance": "...", "Stress": "...", "Reframe": "..." },
    "Discern": { "Test": "...", "Listen": "...", "Filter": "..." },
    "Decide": { "Calibrate": "...", "Posture": "monitor", "Voice": "..." }
  },
  "Posture": "monitor",
  "CounterCase": "The strongest opposing case...",
  "WatchFor": "The condition that would weaken the read...",
  "WhatWouldChangeTheRead": ["late injury status change", "market move against the read"],
  "EvidenceRichness": 2,
  "ArtifactQualityWarnings": []
}
```

it is an unstructured string. there is no `sections[]`, no `brief_id`, no `delivery_status`, and no stored competition metadata inside the output payload itself. the shape is whatever the service layer composes and serializes — the .NET layer does not enforce a schema at the database column level.

**lean field:** optional (`string? / null`). present when the model successfully emits a directional signal. absent (null) when the model does not include it or the value is empty. stored as-is in OutputJson and returned in the response DTO.

**cognitive artifact fields:** `cognitivePhases` is the internal 4-phase artifact from FastAPI and is stored in OutputJson only. `posture`, `counterCase`, `watchFor`, `whatWouldChangeTheRead`, and `evidenceRichness` are compact delivery fields that can be returned in `AgentRunResultDto`. `evidenceRichness` is nullable: null means an older record predates the field, while 0 means the current retriever found no grounded signals.

**artifact quality fields:** `MissingSignals` and `ArtifactQualityWarnings` are internal quality-loop fields stored in OutputJson. they are available through the artifact inspection endpoint below, not the normal `AgentRunResultDto`.

---

## database entity: AgentRunOutcome

file: `DevCore.Domain/Agentic/AgentRunOutcome.cs`
table: `AgentRunOutcomes` (via EF Core, `DevCore.Data/AppDbContext.cs`)
migration added: `20260424182112_AddAgentRunOutcome`

this is the raw real-world result of a game. it is NOT the decision (that is AgentRun+OutputJson). it is NOT a derived evaluation. it is only the factual result: who won, what the score was, when it was recorded, and where it came from.

**one-to-zero-or-one with AgentRun.** a unique index on `AgentRunKey` enforces this at the database level. if an outcome was recorded incorrectly, it is updated in place, not replaced.

| column | type | notes |
|---|---|---|
| AgentRunOutcomeKey | bigint | internal PK, identity |
| AgentRunOutcomeId | guid | public-facing identifier, sequential (`newsequentialid()`) |
| AgentRunKey | bigint | FK to AgentRuns, NoAction on delete; unique index enforces 1:0..1 |
| OutcomeStatus | string(32) | check constraint: `home_win \| away_win \| draw \| cancelled \| postponed` |
| HomeScore | int? | null if not recorded or not applicable (cancelled/postponed) |
| AwayScore | int? | null if not recorded or not applicable |
| Source | string(64) | `"manual"` today; future: `"odds_api"`, `"espn"`, etc. |
| SourceRef | string?(512) | opaque reference to the source event; null for manual entries |
| ResolvedUtc | datetimeoffset | when the outcome was confirmed; server-set unless caller supplies a back-fill timestamp |
| Notes | string?(1000) | free-text; for corrections, postponement details, or anything that doesn't fit above |

**separation rationale:** three things are intentionally kept separate:
1. decision-time snapshot → `AgentRun` + `OutputJson`
2. raw real-world outcome → `AgentRunOutcome`
3. derived run evaluation (was the lean correct?) → future work, not yet implemented

do not collapse these into one field or one record. the learning loop compares (1) against (2) to produce (3).

---

## database entity: AgentRunEvaluation

file: `DevCore.Domain/Agentic/AgentRunEvaluation.cs`
table: `AgentRunEvaluations` (via EF Core, `DevCore.Data/AppDbContext.cs`)
migration added: `20260425145646_AddAgentRunEvaluation`

this is the derived judgment after comparing the decision-time lean to the real-world outcome. it is NOT the decision (AgentRun+OutputJson). it is NOT the raw outcome (AgentRunOutcome). it is only the interpretation: was the lean directionally correct?

**computed at outcome recording time**, in the same transaction as AgentRunOutcome. no separate endpoint. no background job.

**one-to-zero-or-one with AgentRun.** unique index on `AgentRunKey`. "unresolved" is never stored as a string — it is expressed by the absence of this record.

**evaluation logic** lives in `RunEvaluator` (static class, `DevCore.Api/AgentRuns/RunEvaluator.cs`). pure computation, no I/O.

| column | type | notes |
|---|---|---|
| AgentRunEvaluationKey | bigint | internal PK, identity |
| AgentRunEvaluationId | guid | public-facing identifier, sequential |
| AgentRunKey | bigint | FK to AgentRuns, NoAction on delete; unique index enforces 1:0..1 |
| EvalStatus | string(16) | check constraint: `correct \| incorrect \| inconclusive` |
| LeanSide | string?(8) | snapshotted from `AgentRun.LeanSide` at evaluation time; "home", "away", or null |
| WinningSide | string?(8) | derived from `OutcomeStatus` at evaluation time; "home", "away", or null |
| EvaluatedUtc | datetimeoffset | always server-set at outcome recording time |

**EvalStatus logic:**
- `correct` — leanside and winning side match
- `incorrect` — leanside and winning side are opposite
- `inconclusive` — leanside is null, or outcome was draw/cancelled/postponed (no directional winner)

**LeanSide note:** `AgentRun.LeanSide` comes from `AgentRunExecutionResult.LeanSide`, which comes from `SportsAnalysisResponse.LeanSide` (FastAPI). FastAPI now returns `lean_side` as part of every analyze response. The field is normalized to only "home", "away", or null — invalid model output is clamped to null and logged. The `[JsonPropertyName("lean_side")]` attribute on the C# record handles the snake_case → PascalCase mapping at deserialization.

**indexes:**
- `IX_AgentRunEvaluations_AgentRunKey` (unique) — enforces 1:0..1, run-to-eval lookup
- `IX_AgentRunEvaluations_EvalStatus_EvaluatedUtc` — calibration queries: accuracy by status over time

---

## API contract: record an outcome

**endpoint:** `POST /api/agent-runs/{agentRunId}/outcome`
**file:** `DevCore.Api/Controllers/AgentRunsController.cs`

scoped to tenant only (not user) — future automated feeds won't have a user key.

### request body

```json
{
  "outcomeStatus": "home_win",
  "homeScore": 27,
  "awayScore": 17,
  "source": "manual",
  "sourceRef": null,
  "notes": null,
  "resolvedUtc": null
}
```

`resolvedUtc` is optional. if omitted, the server uses `UtcNow`. supply a past timestamp when back-filling historical outcomes.

### responses

| status | condition |
|---|---|
| 201 Created | outcome recorded; body is `AgentRunOutcomeDto` |
| 404 Not Found | run does not exist for this tenant |
| 409 Conflict | an outcome has already been recorded for this run |
| 422 Unprocessable | `outcomeStatus` is not a known value |

### response body (201)

```json
{
  "agentRunOutcomeId": "...",
  "agentRunId": "...",
  "outcomeStatus": "home_win",
  "homeScore": 27,
  "awayScore": 17,
  "source": "manual",
  "sourceRef": null,
  "resolvedUtc": "...",
  "notes": null
}
```

`agentRunId` is included so the caller can cross-reference without a separate lookup.

---

## API contract: create an agent run

**endpoint:** `POST /api/agent-runs`
**file:** `DevCore.Api/Controllers/AgentRunsController.cs`
**auth:** no `[Authorize]` attribute on the controller or this action — identity resolution runs via `IdentityResolver`, which uses dev bypass when `Dev:EnableBypassAuth = true` (currently true in `appsettings.Development.json`)

### request body

```json
{
  "runType": "sports.matchup.analysis",
  "agentProfileId": null,
  "input": {
    "competition": "nfl",
    "homeTeam": "Kansas City Chiefs",
    "awayTeam": "Buffalo Bills",
    "gameDate": "2026-04-13"
  }
}
```

`agentProfileId` is accepted but not used. no profile is loaded or validated during execution.

### response body (success)

```json
{
  "agentRunId": "...",
  "status": "completed",
  "lean": "Edge toward Chiefs based on rest advantage and line movement since open.",
  "summary": "...",
  "confidence": 0.64,
  "factors": ["Market: Current spread supports the lean.", "Rest: Chiefs have the schedule edge."],
  "createdUtc": "...",
  "durationMs": 1842,
  "posture": "monitor",
  "counterCase": "The strongest opposing case...",
  "watchFor": "The condition that would weaken the read...",
  "whatWouldChangeTheRead": ["late injury status change", "market move against the read"],
  "evidenceRichness": 2
}
```

`lean` is positioned before `summary` to reflect decision-first ordering: the directional signal leads, the fuller explanation follows. `lean` may be null if the model response did not include it.
The UI labels `posture` as `Read Stance`, never `Pick`.
Allowed posture values are `play`, `pass`, `monitor`, `wait`, `compare`, and `avoid`.
`evidenceRichness` is from the retriever's grounded signal count, not from model output.

### response body (failure)

status is `"failed"`, summary and factors are null or absent. `ErrorMessage` is stored on the AgentRun row but is not currently returned in the response DTO.

for analyze-stage failures specifically, `AgentRunsController` now catches `AnalysisPipelineException`, marks the row `failed`, stores `ErrorMessage` from the wrapped inner exception, and persists `FailureArtifact` into `OutputJson` before re-throwing. that means failed analyze runs now keep:

- `publishability`
- `degradationNotes`
- `pipelineSteps`

inside the stored output payload instead of leaving `OutputJson` as the placeholder `{}`.

---

## API contract: get an agent run

**endpoint:** `GET /api/agent-runs/{agentRunId}`
**file:** `DevCore.Api/Controllers/AgentRunsController.cs`

response is `AgentRunDetailDto` — includes all fields from the result DTO plus the raw `InputJson` and `OutputJson` strings. tenant and user scoping is applied to the query — a run from a different tenant cannot be retrieved.

---

## API contract: inspect an agent run artifact

**endpoint:** `GET /api/agent-runs/{agentRunId}/artifact`
**file:** `DevCore.Api/Controllers/AgentRunsController.cs`

response is `AgentRunArtifactDto` -- a curated, read-only inspection surface for internal artifact quality and cognitive phase review.
it is for platform learning and debugging, not the main user-facing sports read.
tenant and user scoping matches the existing detail endpoint.

the response includes:
- run metadata: `agentRunId`, `status`, `competition`, `gameDate`
- evidence fields: `groundedSignals`, `missingSignals`, `evidenceRichness`
- quality fields: `artifactQualityWarnings`, `pipelineSteps`
- cognitive fields: `cognitivePhases`, `analyzerConfidence`, `confidence`
- compact delivery fields: `posture`, `counterCase`, `watchFor`, `whatWouldChangeTheRead`, `summary`, `lean`

the endpoint returns 404 when the run is not visible to the caller or when `OutputJson` is missing, empty, or cannot be deserialized into the current artifact contract.
it does not expose raw prompts, provider metadata, api keys, or unrelated tenant/user data.

Dev Artifact Review Page v1 in `apps/sports-app` consumes this endpoint at `/dev/artifacts`.
It provides an internal review surface for inspecting completed AgentRun artifacts through the existing artifact endpoint.
It is for builder learning, debugging, and quality review, not the main user-facing sports read.
explicit admin/dev role gating is deferred until the app has a role model.

Dev Artifact Review Selector v1 adds `GET /api/agent-runs/recent` — a tenant/user-scoped read of recent sports matchup analysis runs.
default take is 25, capped at 50.
returns `AgentRunRecentDto` with denormalized columns plus home/away teams parsed from InputJson.
no OutputJson fields are included in the list.

Dev Upcoming Artifact Runs v1 adds `GET /api/competitions/{code}/upcoming?days=7` in `SportsReferenceController`.
returns `MatchupEventDto[]` (date + homeTeam + awayTeam) for all events in the competition within the given window (default 7 days, max 14).
no team-pair filtering. backed by `OddsScheduleClient.GetAllUpcomingEventsAsync`. 30-minute cache.
intended only for the `/dev/artifacts` batch sample tool, not for production pair-based lookups.

---

## internal contracts: .NET to FastAPI

**file:** `DevCore.AiClient/SportAnalysisContracts.cs`

request sent to FastAPI:

```
SportsAnalysisRequest {
  Competition,
  HomeTeam,
  AwayTeam,
  GameDate,
  FootballMarketContext?,
  MlbStarterContext?,
  BasketballScheduleContext?,
  BasketballMarketContext?,
  SharpPublicContext?
}
```

response received from FastAPI:

```
SportsAnalysisResponse {
  Lean?,
  LeanSide?,
  SignalsUsed?,
  Summary,
  Confidence,
  Factors[],
  Phases?,
  Posture?,
  CounterCase?,
  WatchFor?,
  WhatWouldChangeTheRead?
}
```

these are the only contracts exchanged between .NET and FastAPI. FastAPI validates `lean_side`, `signals_used`, `posture`, and the cognitive phase shape before .NET receives the response. .NET owns evidence richness separately through `SportsRetrievalOutput.GroundedSignals`.

---

## cross-layer correlation note

two ids exist in the flow today. they are different values and should not be confused:

| id | value | stored where |
|---|---|---|
| `AgentRun.CorrelationId` | `HttpContext.TraceIdentifier` — ASP.NET short trace id (e.g. `0HNCK4FSIM14L:00000001`) | `AgentRuns` table |
| `X-Correlation-Id` header to FastAPI | `Activity.Current?.Id` — W3C trace format (e.g. `00-abc...def-01`) | FastAPI logs only |
| `X-Agent-Run-Id` header to FastAPI | `AgentRun.AgentRunId` — stable public GUID | FastAPI logs only |

`X-Agent-Run-Id` is the recommended cross-layer tracing anchor. it links a FastAPI log entry directly to the `AgentRuns` table row with no format mismatch.

---

## RunType dispatch

`AgentRunService.ExecuteAsync` now dispatches by `RunType` using a switch expression.

**known run types** are defined in `RunTypes` (static class in `AgentRunContracts.cs`):

| constant | string value | handler |
|---|---|---|
| `RunTypes.SportsMatchupAnalysis` | `"sports.matchup.analysis"` | `ExecuteSportsMatchupAsync` (private method) |

**unknown run types** throw `NotSupportedException("run type '...' is not supported")`. the controller catches this as an unhandled exception, marks the `AgentRun` row `failed`, stores the message in `ErrorMessage`, and re-throws (resulting in a 500 response to the client).

**analyze-stage failures** are different from generic failures: `AgentRunService` wraps them in `AnalysisPipelineException` with a composed not-publishable failure artifact. the controller persists that artifact to `OutputJson` before re-throwing so the run record preserves pipeline-step and publishability metadata for debugging and the future learning loop.

**input contract note:** `CreateAgentRunRequest.Input` is currently typed as `CompetitionMatchupInput` — still matchup-analysis-specific, but no longer overloaded as a generic "sport" input. when a second run type is introduced, `Input` will still need to become a generic envelope (for example `JsonElement`) and each run type will own its own input shape.

---

## what is not enforced today

- `AgentProfileKey` — accepted in the request, stored as nullable on the row, but no profile is loaded or used during execution
- `OutputJson` schema — no validation; any JSON string FastAPI returns is stored and returned
- `CognitivePhases` database shape — stored inside OutputJson, not split into queryable columns
- auth on `AgentRunsController` — no `[Authorize]` attribute; production enforcement would require adding it and removing the dev bypass
- `CreateAgentRunRequest.Input` typed as `CompetitionMatchupInput` — still run-type-specific and will need generalization before a second run type can be added
