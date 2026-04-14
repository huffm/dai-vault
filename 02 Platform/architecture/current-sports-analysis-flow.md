# current sports analysis flow

**date:** 2026-04-14
**derived from:** actual code in `apps/sports-app/`, `platform/dotnet/`, and `services/agent-service/`
**status:** reflects what is implemented today — not a design target

---

## supported sports

NFL, NBA, and MLB. each sport has its own system prompt in `sports_analyzer.py`. the response contract is identical across all three: `{ summary, confidence, factors[] }`.

---

## selection model: three states

the Angular selection flow is intentionally divided into three distinct states. keeping them separate is the reason the UX can be order-independent while the analysis payload remains correctly oriented.

| state | what it holds | where it lives |
|---|---|---|
| **neutral selection** | `teamA`, `teamB` — the user's chosen pair, no orientation implied | `app.ts` form controls + `teamAValue`/`teamBValue` signals |
| **selected event** | `MatchupEventDto { date, homeTeam, awayTeam }` — the real scheduled event from the provider, with correct orientation | `selectedEvent` signal, set when a date pill is clicked |
| **analysis payload** | `SportsMatchupInput { sport, homeTeam, awayTeam, gameDate }` — fully resolved, passed to the backend | built in `analyze()` from `selectedEvent()` |

`homeTeam`/`awayTeam` appear only at the analysis payload step. everything before that uses `teamA`/`teamB`.

---

## end-to-end request path

```
Angular sports-app
  → GET /api/sports                                  (load active sport list from SQL)
  → GET /api/sports/{slug}/teams                     (load teams for selected sport from SQL)
  → GET /api/sports/{slug}/matchup-dates             (non-directional: ?teamA=X&teamB=Y)
      → OddsScheduleClient.GetEventsAsync()          (matches either home/away orientation)
      ← MatchupEventDto[] { date, homeTeam, awayTeam } (real orientation from provider)
  [user selects a date pill → selectedEvent resolved from matchupEvents]
  → POST /api/agent-runs                             (payload uses selectedEvent.homeTeam/awayTeam)
    → AgentRunsController.Create()
      → IdentityResolver (dev bypass: TenantKey=1, UserKey=1)
      → AppDbContext: INSERT AgentRun (status=pending)
      → AgentRunService.ExecuteAsync()
        → FastApiClient.AnalyzeSportsMatchupAsync()
          → POST http://127.0.0.1:8000/api/sports/analyze  (X-Correlation-Id header attached)
            → app/routes/sports.py: validates sport, logs correlation_id + matchup details
            → app/services/sports_analyzer.py: calls gpt-4o-mini with sport-specific prompt
          ← SportsAnalysisResponse { summary, confidence, factors }
        ← AgentRunExecutionResult { Summary, Confidence, Factors }
      → AppDbContext: UPDATE AgentRun (status=completed, OutputJson, CompletedUtc, DurationMs)
      ← AgentRunResultDto { agentRunId, status, summary, confidence, factors, createdUtc }
    ← HTTP 200
  ← render summary, confidence (percent), factors (chips with formatted category labels)
```

---

## layer-by-layer detail

### Angular: `apps/sports-app/`

files: `app.ts`, `app.html`, `sports-api.service.ts`, `core/models/agent-run.model.ts`, `team-picker/team-picker.component.ts`

**selection model — teamA / teamB:**
- the matchup selection stage uses a neutral pair model: `teamA` and `teamB` — not home/away
- `homeTeam`/`awayTeam` exist only in the final analysis payload (`SportsMatchupInput`), populated from the resolved event after date selection
- form controls: `sport`, `teamA`, `teamB`, `gameDate` — all required for the Analyze button to enable
- `teamAValue` and `teamBValue` are signals that mirror the form control values; computed properties depend on these for reactivity

**team pickers (`TeamPickerComponent`):**
- thin standalone component; `@Input()` for `options`, `label`, `exclude`, `disabled`, `value`; `@Output()` for `selected` (emits team name) and `cleared`
- parent (`app.ts`) owns form controls and calls `setValue()` in picker event handlers
- internal state: `filterText` signal, `open` signal, `filteredOptions` computed
- selected state: shows name chip with × clear button; unselected: text input with filtered dropdown
- each picker excludes the other's current value from its option list

**matchup events and date resolution:**
- on both teams set: calls `SportsApiService.getMatchupEvents(sport, teamA, teamB)` → `GET /api/sports/{slug}/matchup-dates?teamA=X&teamB=Y`
- response is `MatchupEventDto[]` — each item carries `{ date, homeTeam, awayTeam }` with the resolved home/away orientation from the schedule provider
- `matchupEvents` signal stores the events; `dates` computed derives the date list from them for the pill display
- `selectedEvent` signal is set when a date pill is clicked; it holds the full event including resolved orientation
- `contextLine` computed shows `awayTeam @ homeTeam` using the resolved event orientation (not the user's selection order)

**analysis call:**
- `analyze()` reads `selectedEvent()` for `homeTeam`, `awayTeam`, and `gameDate` — the user's Team A/B order is irrelevant here
- `SportsMatchupInput` is unchanged: `{ sport, homeTeam, awayTeam, gameDate }`

**factor label casing:**
- `formatCategory(raw: string)` method capitalizes the first letter after each space or `/`: `"rest/fatigue"` → `"Rest/Fatigue"`, `"injury report"` → `"Injury Report"`
- factor chips split on the first `: ` in the template; the left side (category) is rendered through `formatCategory()`; the right side (observation) is rendered as plain text
- fallback: factors without `: ` render as a single unsplit chip (maintains backward compatibility)

**other:**
- on submit: form disabled with `{ emitEvent: false }`; re-enabled on success or error
- `useStubSchedule = false` — stub path still works: generates `MatchupEventDto[]` with teamA as home
- `environment.useStubApi = false` in development — real API is called
- **no bearer token is attached** — no MSAL configuration or HTTP interceptor

### .NET: `DevCore.Api/Controllers/SportsReferenceController.cs`

- `GET /api/sports` — returns active sports ordered by SportKey (NFL first): `[{ slug, displayName }]`
- `GET /api/sports/{slug}/teams` — returns 404 if slug is unknown or inactive; returns 200 with `[]` if sport has no active teams; otherwise returns `[{ name }]` ordered alphabetically
- `GET /api/sports/{slug}/matchup-dates?teamA=X&teamB=Y` — validates sport (404), validates both teams exist and are active for that sport (400), validates teamA ≠ teamB (400); calls `OddsScheduleClient.GetEventsAsync()` and returns `MatchupEventDto[]`; empty array means no game in the 60-day window
- no auth, no tenant scope — platform-level reference data
- `MatchupEventDto(string Date, string HomeTeam, string AwayTeam)` is defined in `DevCore.Api.Sports` namespace (`OddsScheduleClient.cs`)

### .NET: `DevCore.Api/Sports/OddsScheduleClient.cs`

typed `HttpClient` that calls `api.the-odds-api.com` to look up scheduled matchup events. owned by .NET because schedule data is reference data — it does not feed into prompts and does not require AI processing.

- `GetEventsAsync(sport, teamA, teamB)` — non-directional: matches events in either orientation; cache key uses alphabetically sorted team names so `(A,B)` and `(B,A)` share a cache entry; 30-minute TTL; empty results are cached
- on cache miss: calls Odds API `GET /v4/sports/{oddsKey}/events` with 60-day window; matches `(home==teamA && away==teamB) || (home==teamB && away==teamA)`
- returns `MatchupEventDto[]` — each item carries the real home/away orientation from the provider, not the caller's input order
- dates extracted in Eastern timezone (not UTC)
- sport slug → Odds API key map: `nfl→americanfootball_nfl`, `nba→basketball_nba`, `mlb→baseball_mlb`
- **team name normalization:** verified mismatches as of 2026-04-13: `"Oakland Athletics"` / `"Sacramento Athletics"` → `"Athletics"`
- on network failure or non-2xx: logs error body at `LogError`, returns `[]`

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
- uses `HttpRequestMessage` / `SendAsync` per call (not `DefaultRequestHeaders`) — required because typed HttpClient instances are singletons; per-request headers must not be shared state
- attaches `X-Correlation-Id` header on every call: `Activity.Current?.Id ?? Guid.NewGuid().ToString()`
- on non-2xx response: logs response body at `LogError` (truncated to 500 chars) before throwing
- `ILogger<FastApiClient>` injected via constructor

### FastAPI: `services/agent-service/`

app module lives under `services/agent-service/app/`. entry point is `services/agent-service/main.py` (not inside `app/`), which mounts the sports router at `/api`.

- `app/routes/sports.py`: validates sport against `{"nfl", "nba", "mlb"}`; returns 400 with message for any other value; logs `correlation_id`, `sport`, `homeTeam`, `awayTeam`, `gameDate` at INFO on every request; dispatches to the correct analyzer function
- `app/services/sports_analyzer.py`: one async function per sport (`analyze_nfl`, `analyze_nba`, `analyze_mlb`); each uses a sport-specific system prompt; all call `gpt-4o-mini` with `response_format=json_object` and `temperature=0.3`; shared `_call_model` and `_parse_response` helpers avoid duplication; confidence is clamped 0.0–1.0 in `_parse_response`
- `app/models/sports.py`: Pydantic models `SportsAnalysisRequest` and `SportsAnalysisResponse`

**factor format:** factors are structured as `"category: brief observation"` — not freeform phrases. sport-specific categories:
- NFL: line movement, injury report, situational, weather (outdoor only), sharp/public
- NBA: rest/fatigue, injury report, matchup style, home court — weather never mentioned
- MLB: starting pitching, bullpen, lineup form, ballpark

example verified output (NBA): `["rest/fatigue: Miami on second night of back-to-back", "injury report: no significant injuries for either team", "matchup style: Miami plays slower pace than Charlotte", "home court: Charlotte has a strong home record"]`

---

## SQL reference data

two platform-level tables, not tenant-scoped:

**Sports** — 3 rows: NFL (key=1), NBA (key=2), MLB (key=3). `IsActive` flag controls visibility.

**Teams** — 92 rows: 32 NFL, 30 NBA, 30 MLB. `Name` is the display label and the value sent to the analyzer. `IsActive` flag controls visibility.

team names are the source of truth for what reaches the analyzer. the UI cannot submit a team name that is not in the database.

**note on Athletics:** the DB currently stores `"Oakland Athletics"` (the `RenameOaklandToSacramento` migration has not been applied to the running instance). `OddsScheduleClient._nameMap` covers both names defensively.

---

## what is stored after a run

`AgentRun` row in SQL Server:

| field | value after a successful sports run |
|---|---|
| RunType | `"sports.matchup.analysis"` |
| Status | `"completed"` |
| InputJson | `{"sport":"nfl","homeTeam":"...","awayTeam":"...","gameDate":"..."}` |
| OutputJson | `{"summary":"...","confidence":0.72,"factors":["category: observation","..."]}` |
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

`OddsApi:ApiKey` must be set in user secrets for matchup-dates to return real data: `dotnet user-secrets set "OddsApi:ApiKey" "your-key-here"` from `platform/dotnet/DevCore.Api/`.

---

## product rule: outward-facing UI text must be properly cased

raw values from the backend (factor categories, sport slugs, status strings) are lowercase by convention for internal consistency. before rendering in the UI, outward-facing text is formatted:

- **factor categories:** `formatCategory()` in `app.ts` capitalizes the first letter after each space or `/`. `"rest/fatigue"` → `"Rest/Fatigue"`. `"injury report"` → `"Injury Report"`. applied to the category portion of factor chips only.
- **status:** Angular `titlecase` pipe applied to `r.status` in the result panel tile.
- **team names and sport names:** stored with correct casing in SQL; rendered as-is.

this rule applies to all outward-facing product text. internal code, logs, API contracts, and config values stay lowercase.

---

## current limitations

| limitation | location | impact |
|---|---|---|
| no auth on sports path | `AgentRunsController`, `SportsApiService` | any caller can create runs; no tenant scoping in production |
| production `useStubApi` | `apps/sports-app/environment.ts` (`useStubApi: true`) | `createMatchupAnalysis()` returns a hardcoded stub response in production; real API is only called when `useStubApi: false` (dev) |
| OutputJson has no schema | `AgentRun.OutputJson` | stored content cannot be queried or validated structurally |
| AgentProfileId accepted but ignored | `AgentRunService` | profile-level prompt configuration is not applied |
| no .NET sport gate | `AgentRunService` | invalid sports are caught at FastAPI (400), not at the .NET layer; failed runs still write an AgentRun row |
| RenameOaklandToSacramento migration not applied | running Docker DB | DB has "Oakland Athletics"; normalization map covers both names but migration should be applied before a real deployment |

---

## what is not in this flow

these concepts appear in vault docs but have no implementation in this path today:

- signal scoring (+1 / 0 / -1 / null)
- collector, evaluator, synthesizer, compliance stages
- external data sources beyond schedule dates (actionnetwork, rotowire, open-meteo)
- delivery (email, webhook, slack)
- scheduled triggers
- tier enforcement
- stripe
- brief envelope with sections[]
