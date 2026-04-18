# current sports analysis flow

**date:** 2026-04-18
**derived from:** actual code in `apps/sports-app/`, `platform/dotnet/`, and `services/agent-service/`
**status:** reflects what is implemented today — not a design target

---

## supported sports

NFL, NBA, and MLB. each sport has its own system prompt in `sports_analyzer.py`. the backend response is `{ lean, summary, confidence, factors[] }`. lean is an optional string — `"edge toward [team] based on [reason]"` or `"signals are split"` — emitted by the model and propagated through all contract layers to the Angular `Current Lean` module.

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
          [nfl only] → OddsMarketClient.GetNflSpreadAsync()
                         → GET /v4/sports/americanfootball_nfl/odds?regions=us&markets=spreads&...
                         ← NflMarketContext { HomeTeam, AwayTeam, HomeSpread, AwaySpread, Bookmaker, UpdatedAt }
                            or null (offseason, api error, no matching event)
        → FastApiClient.AnalyzeSportsMatchupAsync(req with NflMarket?)
          → POST http://127.0.0.1:8000/api/sports/analyze  (X-Correlation-Id + X-Agent-Run-Id headers attached)
            → app/routes/sports.py: validates sport, logs agent_run_id + correlation_id + matchup details
            → app/services/sports_analyzer.py:
                [nfl] builds user message with [market data] block if NflMarketContext present
                      injects "omit market factor" instruction if no context
                calls gpt-4o-mini with sport-specific system prompt + user message
          ← SportsAnalysisResponse { lean, summary, confidence, factors }
        ← AgentRunExecutionResult { Lean, Summary, Confidence, Factors }
      → AppDbContext: UPDATE AgentRun (status=completed, OutputJson, CompletedUtc, DurationMs)
      ← AgentRunResultDto { agentRunId, status, lean, summary, confidence, factors, createdUtc, durationMs }
    ← HTTP 200
  ← render page surfaces:
      hero shell
      top row = Configure Matchup (left) + Matchup Read (right)
      below row = Factor Breakdown (full-width supporting reasoning)
```

---

## layer-by-layer detail

### Angular: `apps/sports-app/`

files: `app.ts`, `app.html`, `sports-api.service.ts`, `core/models/agent-run.model.ts`, `team-picker/team-picker.component.ts`, `team-picker/team-picker.component.scss`

**selection model — teamA / teamB:**
- the matchup selection stage uses a neutral pair model: `teamA` and `teamB` — not home/away
- `homeTeam`/`awayTeam` exist only in the final analysis payload (`SportsMatchupInput`), populated from the resolved event after date selection
- form controls: `sport`, `teamA`, `teamB`, `gameDate` — all required for the Analyze button to enable
- `teamAValue` and `teamBValue` are signals that mirror the form control values; computed properties depend on these for reactivity

**picker family (`TeamPickerComponent`):**
- single standalone picker component used for all three controls: sport, team a, and team b
- `sport` uses the picker with `searchable=false`; `teamA` and `teamB` use `searchable=true`
- `@Input()` surface: `options`, `label`, `exclude`, `disabled`, `value`, `searchable`, `placeholder`, `searchPlaceholder`
- `@Output()` surface: `selected` (emits option value) and `cleared`
- parent (`app.ts`) owns form controls and calls `setValue()` in picker event handlers
- internal state: `filterText` signal, `open` signal, `filteredOptions` computed
- selected state: shows a chip-style selected shell with a clear button; unselected state is either a searchable input or a plain trigger button depending on `searchable`
- dropdowns expose listbox semantics and keyboard-friendly button options; escape closes the open menu
- each team picker excludes the other's current value from its option list
- `team-picker.component.scss` sets `:host { display: block }` so the custom element participates in block layout and `space-y-4` spacing is consistent with adjacent `<div>` field groups

**matchup events and date resolution:**
- on both teams set: calls `SportsApiService.getMatchupEvents(sport, teamA, teamB)` → `GET /api/sports/{slug}/matchup-dates?teamA=X&teamB=Y`
- response is `MatchupEventDto[]` — each item carries `{ date, homeTeam, awayTeam }` with the resolved home/away orientation from the schedule provider
- `matchupEvents` signal stores the events; `dates` computed derives the date list from them for the pill display
- `selectedEvent` signal is set when a date pill is clicked; it holds the full event including resolved orientation
- `contextLine` computed shows `awayTeam @ homeTeam` using the resolved event orientation (not the user's selection order)

**analysis call:**
- `analyze()` reads `selectedEvent()` for `homeTeam`, `awayTeam`, and `gameDate` — the user's Team A/B order is irrelevant here
- `SportsMatchupInput` is unchanged: `{ sport, homeTeam, awayTeam, gameDate }`

**page structure:**
- hero shell sits above the working area
- main row uses a left control rail and a right answer card
- `Matchup Read` is the primary answer card: analyzed matchup, current lean slot, summary, status, confidence
- `Factor Breakdown` is a full-width supporting reasoning section below the row
- the page is intentionally single-page scroll; the app does not hide answer/reasoning behind tabs
- no filler cards are added just to make the page look more square
- sticky behavior is confined to the control rail row so pills and dropdown surfaces do not overlap lower sections during scroll
- the atmospheric background is now owned by a fixed `body::before` layer so longer populated pages do not show seams or darker restart bands

**state model:**
- `resultTone()` drives the inner analysis shell across idle, loading, error, and completed states
- idle state renders an anchored inner empty-state panel inside `Matchup Read`
- loading state renders skeleton blocks inside the same shell
- error state stays in-place with fixed product copy
- completed state renders analyzed matchup, current lean, summary, status/confidence, diagnostics, and factor tiles
- hero chips are computed from selected sport, teams, and resolved event/date; the `dev` chip appears only when stub schedule mode is active

**run metadata visibility:**
- `agentRunId` is now surfaced in the Analysis Details block (left control rail) as the first 8 characters in monospace, with the full UUID on hover via `title` attribute
- `durationMs` is now surfaced as a formatted "Run time" row (e.g., `1.2s` or `850ms`); conditional — only shown when the backend provides a non-null value
- these appear in the Analysis Details block below Signal quality, using the same label: value row pattern
- the correlation path `agentRunId` → `AgentRun.AgentRunId` → `X-Agent-Run-Id` in FastAPI logs is now usable without DevTools

**current lean slot:**
- the `Current Lean` module in `Matchup Read` is now populated from the backend
- FastAPI emits `lean` as a one-sentence directional signal: `"edge toward [team] based on [reason]"` or `"signals are split"`
- lean flows through `SportsAnalysisResponse → AgentRunExecutionResult → AgentRunResultDto` at every layer
- `AgentRunResultDto` in the frontend has `lean?: string | null`; the `currentLean()` computed renders it as the primary text in the lean module
- lean is optional throughout — if the model omits it or returns an empty string, `_parse_response` normalizes to `None` and the frontend falls back to "Lean unavailable"
- the UI does not infer a pick from `summary` or `factors`; lean is the only directional output

**sport display rule:**
- internal sport values remain lowercase slugs (`nfl`, `nba`, `mlb`)
- outward-facing UI paths resolve the label from `SportDto.displayName`
- if lookup fails unexpectedly, the fallback still avoids leaking a raw lowercase slug into the UI

**factor label casing:**
- `formatCategory(raw: string)` method capitalizes the first letter after each space or `/`: `"rest/fatigue"` → `"Rest/Fatigue"`, `"injury report"` → `"Injury Report"`
- factor cards split on the first `: ` in the template; the title uses `formatCategory()` and the body renders the remaining observation text
- fallback: factors without `: ` render with a generic label and the full factor as body copy

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

### .NET: `DevCore.Api/Sports/OddsMarketClient.cs`

typed `HttpClient` that fetches current nfl spread data from the odds api before each nfl analysis. owned in .NET for the same reason as `OddsScheduleClient`: the api key is in .net user secrets, and deterministic data retrieval belongs in the platform layer, not the ai reasoning layer.

- `GetNflSpreadAsync(homeTeam, awayTeam, gameDate, ct)` — returns `NflMarketContext?`
- calls `/v4/sports/americanfootball_nfl/odds?regions=us&markets=spreads&commenceTimeFrom=...&commenceTimeTo=...` bracketed to the game date in eastern time
- bookmaker preference: `draftkings > fanduel > betmgm > caesars > pinnacle > first available`
- spread values formatted as signed strings: `"-3.5"` (favorite) or `"+3.5"` (underdog)
- result cached for 15 minutes per team/date combination
- returns `null` on any failure: network error, api non-2xx, offseason empty response, team name mismatch
- uses same team name normalization map as `OddsScheduleClient` — keep in sync when entries change
- **line movement is not available** via the standard `/odds` endpoint. the historical endpoint (`/v4/historical/sports/.../odds`) would require premium access and snapshot timestamps — explicitly deferred.

### .NET: `DevCore.AiClient/NflMarketContext.cs`

record that carries market spread data from .NET to FastAPI as part of `SportsAnalysisRequest`:
```
NflMarketContext { HomeTeam, AwayTeam, HomeSpread, AwaySpread, Bookmaker, UpdatedAt }
```
defined in `DevCore.AiClient` because it is part of the ai service contract, not a domain type. serialized as `nflMarketContext` (camelCase) by `JsonContent.Create` web defaults.

### .NET: `DevCore.Api/AgentRuns/AgentRunService.cs`

- `ExecuteAsync` dispatches by `req.RunType` using a switch expression
- `RunTypes.SportsMatchupAnalysis` (`"sports.matchup.analysis"`) routes to `ExecuteSportsMatchupAsync`
- unknown run types throw `NotSupportedException` — the controller persists the run as `failed`
- `ExecuteSportsMatchupAsync` now:
  1. if sport is `nfl`: calls `OddsMarketClient.GetNflSpreadAsync()` before building the ai request
  2. builds `SportsAnalysisRequest { Sport, HomeTeam, AwayTeam, GameDate, NflMarket? }` and calls `FastApiClient.AnalyzeSportsMatchupAsync()`
  3. `NflMarket` is null if the odds call failed or returned no data — fastapi falls back gracefully
- known run type strings live in `RunTypes` static class in `AgentRunContracts.cs` — add new entries there when new run types are introduced
- no sport validation at this layer — FastAPI enforces the supported sport list and returns 400 for unknowns

### .NET: `DevCore.AiClient/FastApiClient.cs`

- typed `HttpClient` with base URL from `AiService:BaseUrl` (development: `http://127.0.0.1:8000`)
- timeout: 90 seconds
- uses `HttpRequestMessage` / `SendAsync` per call (not `DefaultRequestHeaders`) — required because typed HttpClient instances are singletons; per-request headers must not be shared state
- attaches `X-Correlation-Id` header on every call: `Activity.Current?.Id ?? Guid.NewGuid().ToString()`
- attaches `X-Agent-Run-Id` header on sports calls: the stable `AgentRun.AgentRunId` GUID — this is the primary cross-layer tracing anchor; FastAPI logs it so any log entry can be tied back to the db row
- on non-2xx response: logs response body at `LogError` (truncated to 500 chars) before throwing
- `ILogger<FastApiClient>` injected via constructor

### FastAPI: `services/agent-service/`

app module lives under `services/agent-service/app/`. entry point is `services/agent-service/main.py` (not inside `app/`), which mounts the sports router at `/api`.

- `app/routes/sports.py`: validates sport against `{"nfl", "nba", "mlb"}`; returns 400 with message for any other value; logs `agent_run_id` (from `X-Agent-Run-Id`), `correlation_id`, `sport`, `homeTeam`, `awayTeam`, `gameDate` at INFO on every request; dispatches to the correct analyzer function
- `app/services/sports_analyzer.py`: one async function per sport (`analyze_nfl`, `analyze_nba`, `analyze_mlb`); each uses a sport-specific system prompt; all call `gpt-4o-mini` with `response_format=json_object` and `temperature=0.3`; shared `_call_model` and `_parse_response` helpers avoid duplication; confidence is clamped 0.0–1.0 in `_parse_response`
- `app/models/sports.py`: Pydantic models `SportsAnalysisRequest` and `SportsAnalysisResponse`

**nfl market enrichment:**
- `analyze_nfl` receives optional `market_context: NflMarketContext | None` from the route handler
- when present: builds a `[market data]` block in the user message — home team, spread, bookmaker, timestamp — and instructs the model to use it for the `market` factor. model cites the real spread without fabricating movement.
- when absent: appends `[no market data available]` to the user message and explicitly instructs the model to omit the market factor and not fabricate spread or line direction.
- the nfl system prompt no longer instructs the model to "note direction, magnitude, and timing" (which caused fabrication). it now says: use only what is in the `[market data]` block.

**factor format:** factors are structured as `"category: brief observation"` — not freeform phrases. sport-specific categories:
- NFL (when market data present): market, injury report, situational, weather (outdoor only)
- NFL (when no market data): injury report, situational, weather (outdoor only) — market factor omitted
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
| DurationMs | real elapsed time from .NET's perspective — now included in the API response DTO |

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

- **factor categories:** `formatCategory()` in `app.ts` capitalizes the first letter after each space or `/`. `"rest/fatigue"` → `"Rest/Fatigue"`. `"injury report"` → `"Injury Report"`. applied to the category portion of factor cards only.
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
- nfl line movement (opening vs current) — standard /odds endpoint does not provide it. requires historical snapshot endpoint (premium) — explicitly deferred
- nba/mlb market enrichment — nfl is the only sport with market context in this slice; nba/mlb prompts are unchanged
- injury data (rotowire) — no integration; model uses prior knowledge only
- sharp/public split (actionnetwork) — no integration; removed from nfl system prompt in this slice to reduce fabrication surface
- weather (open-meteo) — no integration; still mentioned in nfl system prompt as a structural category (model uses general knowledge)
- delivery (email, webhook, slack)
- scheduled triggers
- tier enforcement
- stripe
- brief envelope with sections[]
