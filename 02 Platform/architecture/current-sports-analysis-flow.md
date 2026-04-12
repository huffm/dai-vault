# current sports analysis flow

**date:** 2026-04-11
**derived from:** actual code in `apps/sports-app/`, `platform/dotnet/`, and `services/agent-service/`
**status:** reflects what is implemented today — not a design target

---

## supported sports

NFL, NBA, and MLB. each sport has its own system prompt in `sports_analyzer.py`. the response contract is identical across all three: `{ summary, confidence, factors[] }`.

---

## end-to-end request path

```
Angular sports-app
  → GET /api/sports                          (load active sport list from SQL)
  → GET /api/sports/{slug}/teams             (load teams for selected sport from SQL)
  → GET /api/sports/{slug}/matchup-dates     (load available dates for home+away pairing; or local stub in dev)
  → POST /api/agent-runs
    → AgentRunsController.Create()
      → IdentityResolver (dev bypass: TenantKey=1, UserKey=1)
      → AppDbContext: INSERT AgentRun (status=pending)
      → AgentRunService.ExecuteAsync()
        → FastApiClient.AnalyzeSportsMatchupAsync()
          → POST http://127.0.0.1:8000/api/sports/analyze
            → app/routes/sports.py: validates sport, dispatches to analyzer
            → app/services/sports_analyzer.py: calls gpt-4o-mini with sport-specific prompt
          ← SportsAnalysisResponse { summary, confidence, factors }
        ← AgentRunExecutionResult { Summary, Confidence, Factors }
      → AppDbContext: UPDATE AgentRun (status=completed, OutputJson, CompletedUtc, DurationMs)
      ← AgentRunResultDto { agentRunId, status, summary, confidence, factors, createdUtc }
    ← HTTP 200
  ← render summary, confidence (percent), factors (chips)
```

---

## layer-by-layer detail

### Angular: `apps/sports-app/`

files: `app.ts`, `app.html`, `sports-api.service.ts`, `core/models/agent-run.model.ts`

- on init: calls `GET /api/sports` and populates sport selector; no default selection — user must pick the sport explicitly
- on sport change: calls `GET /api/sports/{slug}/teams`, resets team selectors, populates dropdowns; cascades clear to dates, gameDate, and result
- team selection is pre-populated from SQL reference data — not free text
- home and away team dropdowns filter each other: the opposing team is excluded from each list
- on both-teams-set: calls `getMatchupDates(sport, home, away)`; in dev stub mode this returns a synthetic 14-day window locally; in real mode calls `GET /api/sports/{slug}/matchup-dates?home=X&away=Y` which currently returns `[]` (no schedule source connected yet)
- game date is a dropdown populated from the dates response; shows "no upcoming matchups found" when real endpoint returns `[]`
- on submit: disables the form with `{ emitEvent: false }` to avoid triggering downstream subscriptions; shows an inline button spinner; calls `SportsApiService.createMatchupAnalysis()` → `POST http://localhost:5007/api/agent-runs`; re-enables with `{ emitEvent: false }` on success or error
- after analysis succeeds: form stays fully populated; result panel shows an "Analyzed Matchup" section (sport, matchup, date) above the summary content
- result panel shows an animated skeleton while the request is in flight; replaced by real content on completion
- **utility area (dual-mode):** the space below the Analyze button is driven by an `@if / @else` block keyed on the `analysisDetails` computed. before analysis: a two-line narration + context strip (`narration` computed for phase status, `contextLine` computed for subordinate context — date source or matchup preview). after analysis: a compact Analysis Details block with four rows — Analysis mode, Team source, Date mode, Signal quality — all derived from `result()`, `analyzedMatchup()`, and `isStubSchedule`. clears back to the strip automatically when any upstream field change wipes the result.
- **result diagnostics:** a `resultDiagnostics` computed below the confidence tile in the result panel interprets `result().confidence` as a plain-language sentence. the Signal quality row in the Analysis Details block shows a one-word form of the same confidence band; the two are consistent but serve different reading depths.
- **observable seam for progress events:** in-flight message cycling is driven by `interval(2500)` writing into `analysisPhase` signal; `narration` computed reads `analysisPhase()`. to wire backend progress events: replace `interval(2500)` with an event stream observable, map payloads to phase indices — `narration` computed and the rest of the utility area are unaffected.
- **UI trust constraints:** error states show fixed strings — backend error detail is not forwarded to the UI. no raw run types, agent output, prompts, tool traces, or chain of thought is exposed in any UI surface.
- **no bearer token is attached** — no MSAL configuration or HTTP interceptor
- `environment.useStubApi = false` in development — real API is called
- `environment.useStubSchedule = true` in development — synthetic dates used locally; `false` in production

### .NET: `DevCore.Api/Controllers/SportsReferenceController.cs`

- `GET /api/sports` — returns active sports ordered by SportKey (NFL first): `[{ slug, displayName }]`
- `GET /api/sports/{slug}/teams` — returns 404 if slug is unknown or inactive; returns 200 with `[]` if sport has no active teams; otherwise returns `[{ name }]` ordered alphabetically
- `GET /api/sports/{slug}/matchup-dates?home=X&away=Y` — validates sport (404), validates both teams exist and are active for that sport (400), validates home ≠ away (400); currently returns `[]` — no schedule data source is connected yet; will return real dates when a schedule provider is wired in
- no auth, no tenant scope — platform-level reference data

### .NET: `DevCore.Api/Controllers/AgentRunsController.cs`

- no `[Authorize]` attribute
- `IdentityResolver` runs on every request — dev bypass: TenantKey=1, UserKey=1
- creates an `AgentRun` row with status `pending`, `InputJson` = serialized `SportsMatchupInput`, `StartedUtc` = now
- calls `IAgentRunService.ExecuteAsync()`
- on success: updates row to `completed`, sets `OutputJson`, `CompletedUtc`, `DurationMs`
- on exception: updates row to `failed`, sets `ErrorMessage`
- returns `AgentRunResultDto`

### .NET: `DevCore.Api/AgentRuns/AgentRunService.cs`

- maps `SportsMatchupInput` to `SportsAnalysisRequest { Sport, HomeTeam, AwayTeam, GameDate }`
- calls `FastApiClient.AnalyzeSportsMatchupAsync()`
- no sport validation — FastAPI enforces the supported sport list and returns 400 for unknowns

### .NET: `DevCore.AiClient/FastApiClient.cs`

- typed `HttpClient` with base URL from `AiService:BaseUrl` (development: `http://127.0.0.1:8000`)
- timeout: 90 seconds
- posts to `/api/sports/analyze`, deserializes `SportsAnalysisResponse`

### FastAPI: `services/agent-service/`

app module lives under `services/agent-service/app/`. entry point is `services/agent-service/main.py` (not inside `app/`), which mounts the sports router at `/api`.

- `app/routes/sports.py`: validates sport against `{"nfl", "nba", "mlb"}`; returns 400 with message for any other value; dispatches to the correct analyzer function
- `app/services/sports_analyzer.py`: one async function per sport (`analyze_nfl`, `analyze_nba`, `analyze_mlb`); each uses a sport-specific system prompt; all call `gpt-4o-mini` with `response_format=json_object` and `temperature=0.3`; shared `_call_model` and `_parse_response` helpers avoid duplication; confidence is clamped 0.0–1.0 in `_parse_response`
- `app/models/sports.py`: Pydantic models `SportsAnalysisRequest` and `SportsAnalysisResponse`

---

## SQL reference data

two platform-level tables, not tenant-scoped:

**Sports** — 3 rows: NFL (key=1), NBA (key=2), MLB (key=3). `IsActive` flag controls visibility.

**Teams** — 92 rows: 32 NFL, 30 NBA, 30 MLB. `Name` is the display label and the value sent to the analyzer. `IsActive` flag controls visibility.

team names are the source of truth for what reaches the analyzer. the UI cannot submit a team name that is not in the database.

---

## what is stored after a run

`AgentRun` row in SQL Server:

| field | value after a successful sports run |
|---|---|
| RunType | `"sports.matchup.analysis"` |
| Status | `"completed"` |
| InputJson | `{"sport":"nfl","homeTeam":"...","awayTeam":"...","gameDate":"..."}` |
| OutputJson | `{"summary":"...","confidence":0.72,"factors":["...","...","..."]}` |
| AgentProfileKey | null (not used in this slice) |
| CorrelationId | ASP.NET trace identifier |
| DurationMs | real elapsed time from .NET's perspective |

---

## local dev workflow

scripts live under `scripts/dev/sports/`:

| script | purpose |
|---|---|
| `start-sports-dev.ps1` | launches agent service (port 8000), platform API (port 5007), and sports app (port 4201) in separate terminals |
| `stop-sports-dev.ps1` | stops all three processes by port; does not touch the prompt portal |
| `test-sports-dev.ps1` | five smoke checks: health ping, NFL analysis, stub detection, unsupported sport gate, full .NET chain |

agent service is started with `python -m uvicorn main:app --host 127.0.0.1 --port 8000 --reload`. the `.venv/Scripts/uvicorn.exe` launcher is not used because it contains hardcoded absolute paths that break after a repo move.

smoke tests are part of the normal workflow. run `test-sports-dev.ps1` after starting the stack to confirm all five checks pass before doing further dev work.

---

## current limitations

| limitation | location | impact |
|---|---|---|
| no auth on sports path | `AgentRunsController`, `SportsApiService` | any caller can create runs; no tenant scoping in production |
| production `useStubApi` | `apps/sports-app/environment.ts` (`useStubApi: true`) | `createMatchupAnalysis()` returns a hardcoded stub response in production; real API is only called when `useStubApi: false` (dev) |
| OutputJson has no schema | `AgentRun.OutputJson` | stored content cannot be queried or validated structurally |
| AgentProfileId accepted but ignored | `AgentRunService` | profile-level prompt configuration is not applied |
| no .NET sport gate | `AgentRunService` | invalid sports are caught at FastAPI (400), not at the .NET layer; failed runs still write an AgentRun row |

---

## what is not in this flow

these concepts appear in vault docs but have no implementation in this path today:

- signal scoring (+1 / 0 / -1 / null)
- collector, evaluator, synthesizer, compliance stages
- external data sources (odds api, actionnetwork, rotowire, open-meteo)
- delivery (email, webhook, slack)
- scheduled triggers
- tier enforcement
- stripe
- brief envelope with sections[]
