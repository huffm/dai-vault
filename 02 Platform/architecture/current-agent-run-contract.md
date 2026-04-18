# current agent run contract

**date:** 2026-04-16 (RunType dispatch slice completed)
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

### what OutputJson currently contains

OutputJson is written as the raw serialized `SportsAnalysisResponse` from FastAPI:

```json
{
  "summary": "...",
  "confidence": 0.64,
  "factors": ["...", "...", "..."]
}
```

it is an unstructured string. there is no `sections[]`, no `brief_id`, no `delivery_status`, and no sport identifier in the stored output. the shape is whatever FastAPI returns — the .NET layer does not validate or enforce it.

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
    "sport": "nfl",
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
  "summary": "...",
  "confidence": 0.64,
  "factors": ["...", "...", "..."],
  "createdUtc": "...",
  "durationMs": 1842
}
```

### response body (failure)

status is `"failed"`, summary and factors are null or absent. `ErrorMessage` is stored on the AgentRun row but is not currently returned in the response DTO.

---

## API contract: get an agent run

**endpoint:** `GET /api/agent-runs/{agentRunId}`
**file:** `DevCore.Api/Controllers/AgentRunsController.cs`

response is `AgentRunDetailDto` — includes all fields from the result DTO plus the raw `InputJson` and `OutputJson` strings. tenant and user scoping is applied to the query — a run from a different tenant cannot be retrieved.

---

## internal contracts: .NET to FastAPI

**file:** `DevCore.AiClient/SportAnalysisContracts.cs`

request sent to FastAPI:

```
SportsAnalysisRequest { Sport, HomeTeam, AwayTeam, GameDate }
```

response received from FastAPI:

```
SportsAnalysisResponse { Summary, Confidence, Factors[] }
```

these are the only contracts exchanged between .NET and FastAPI. .NET does not parse or validate the content of `Summary` or `Factors` — it stores them as-is and returns them in the DTO.

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

**input contract note:** `CreateAgentRunRequest.Input` is currently typed as `SportsMatchupInput` — sports-specific fields only. when a second run type is introduced, `Input` will need to become a generic envelope (e.g. `JsonElement`) and each run type will own its own input shape. this is documented in a comment on the record.

---

## what is not enforced today

- `AgentProfileKey` — accepted in the request, stored as nullable on the row, but no profile is loaded or used during execution
- `OutputJson` schema — no validation; any JSON string FastAPI returns is stored and returned
- auth on `AgentRunsController` — no `[Authorize]` attribute; production enforcement would require adding it and removing the dev bypass
- `CreateAgentRunRequest.Input` typed as `SportsMatchupInput` — will need generalization before a second run type can be added
