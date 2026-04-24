# current sports analysis flow

**date:** 2026-04-20  
**derived from:** actual code in `apps/sports-app/`, `platform/dotnet/`, and `services/agent-service/`  
**status:** reflects what is implemented today — not a design target

---

## supported competitions

the product is no longer modeled internally as "sport only". the internal routing key is now an explicit **competition code**.

the current visible matrix is:

| sport family | level | competition code | status |
|---|---|---|---|
| football | pro | `nfl` | supported |
| football | college | `ncaaf` | supported |
| basketball | pro | `nba` | supported |
| basketball | college | `ncaamb` | supported |
| baseball | pro | `mlb` | supported |
| baseball | college | none | shown as unavailable |

user-facing flow stays clearer than raw codes:
- the UI asks for `sport`
- then `level`
- then teams
- then date
- then analyze

the platform resolves that selection to a competition code before teams, dates, retriever routing, or analyzer dispatch happen.

---

## selection model: four states

the Angular selection flow is intentionally split so the UX stays familiar while the backend gets explicit competition identity.

| state | what it holds | where it lives |
|---|---|---|
| **competition selection** | `sportFamily`, `competition` | `AnalyzerComponent` form controls + signals |
| **neutral team selection** | `teamA`, `teamB` — user-selected pair, no orientation implied | `AnalyzerComponent` form controls + `teamAValue`/`teamBValue` |
| **selected event** | `MatchupEventDto { date, homeTeam, awayTeam }` — real scheduled event with resolved orientation | `selectedEvent` signal |
| **analysis payload** | `CompetitionMatchupInput { competition, homeTeam, awayTeam, gameDate }` | built in `analyze()` from `selectedEvent()` |

`homeTeam` and `awayTeam` still appear only at the final payload step. everything before that uses the neutral team pair.

---

## end-to-end request path

```text
Angular sports-app
  → GET /api/competitions
      ← CompetitionReferenceDto[] {
           code, displayName, sportFamily, sportFamilyDisplayName,
           level, levelDisplayName, isSupported, availabilityNote
         }
  [user picks sport family + level]
  → GET /api/competitions/{code}/teams
  → GET /api/competitions/{code}/matchup-dates?teamA=X&teamB=Y
      → OddsScheduleClient.GetEventsAsync()          (non-directional, cached 30min)
      ← MatchupEventDto[] { date, homeTeam, awayTeam }
  [user selects date pill → selectedEvent resolved]
  → POST /api/agent-runs
    payload.input = { competition, homeTeam, awayTeam, gameDate }
    → AgentRunsController.Create()
      → IdentityResolver (dev bypass: TenantKey=1, UserKey=1)
      → INSERT AgentRun (status=pending, CorrelationId=TraceIdentifier)
      → AgentRunService.ExecuteAsync(req, run.AgentRunId, ct)
        → ExecuteSportsMatchupAsync()
          → RetrieveAsync()
            [nfl / ncaaf] OddsMarketClient.GetFootballSpreadAsync()
            [mlb only]  MlbStarterClient.GetStartersAsync()
            [nba / ncaamb] EspnBasketballScheduleClient.GetRestContextAsync()
            [nba / ncaamb] OddsMarketClient.GetBasketballSpreadAsync()
          ← SportsRetrievalOutput { FootballMarketContext?, MlbStarterContext?, BasketballScheduleContext?, BasketballMarketContext?, GroundedSignals[] }
          → FastApiClient.AnalyzeSportsMatchupAsync()
            → POST http://127.0.0.1:8001/api/sports/analyze
                X-Correlation-Id: Activity.Current?.Id
                X-Agent-Run-Id: AgentRun.AgentRunId
              → sports.py validates competition code
              → sports.py dispatches by competition family:
                  nfl / ncaaf    → football analyzer
                  nba / ncaamb   → basketball analyzer
                  mlb            → mlb analyzer
          ← SportsAnalysisResponse { lean, summary, confidence, factors[] }
          → Evaluate(analyzerOutput, retrieval, competition)
          ← EvaluatorOutput { aggregateConfidence, analyzerConfidence, confidenceBand }
      → UPDATE AgentRun (completed, OutputJson, DurationMs)
      ← AgentRunResultDto { agentRunId, status, lean, summary, confidence, factors, createdUtc, durationMs }
```

---

## layer-by-layer detail

### Angular: `apps/sports-app/`

files: `analyzer/analyzer.component.*`, `sports-api.service.ts`, `core/models/agent-run.model.ts`, `team-picker/team-picker.component.*`

**selection model:**
- form controls are now `sportFamily`, `competition`, `teamA`, `teamB`, `gameDate`
- sport family is the first user-facing choice
- level is the second user-facing choice, but internally it selects a concrete competition code
- team pickers stay disabled until a supported competition is selected
- baseball + college is visible as unavailable through the level step, not faked as a working path

**reference data shape:**
- `CompetitionReferenceDto` is now the frontend reference model
- each visible row carries both user-facing labels and internal routing values:
  - `code`
  - `displayName`
  - `sportFamily`
  - `sportFamilyDisplayName`
  - `level`
  - `levelDisplayName`
  - `isSupported`
  - `availabilityNote`

**picker behavior:**
- the same picker family is still used for sport, level, team a, and team b
- `sport` picker uses sport family options in non-searchable mode
- `level` picker uses competition-backed options in non-searchable mode
- unsupported options can render disabled inside the picker
- team pickers remain searchable and exclude the opposite team choice

**date resolution:**
- once both teams are selected, the app calls `GET /api/competitions/{code}/matchup-dates`
- the response is still `MatchupEventDto[]`
- `selectedEvent` still carries the resolved provider orientation
- the user's Team A / Team B ordering still does not control the final `homeTeam` / `awayTeam`

**analysis call:**
- `analyze()` now posts `CompetitionMatchupInput`
- payload shape:

```json
{
  "competition": "ncaaf",
  "homeTeam": "Boston College Eagles",
  "awayTeam": "Syracuse Orange",
  "gameDate": "2026-09-12"
}
```

**hero and result metadata:**
- hero chips now show `sport`, `level`, `matchup`, `date`
- the result card shows sport family and level as separate user-facing rows
- the internal competition code does not leak into the UI unless a fallback is ever needed

### .NET: `DevCore.Api/Controllers/SportsReferenceController.cs`

- `GET /api/competitions`
  - returns the visible competition matrix used by the frontend
  - supported competitions come from active SQL rows
  - unsupported college baseball is appended from `CompetitionCatalog` with `Code = null`, `IsSupported = false`, and `AvailabilityNote = "College baseball is not available yet."`
- `GET /api/competitions/{code}/teams`
  - validates that the competition exists and is active
  - returns active teams for that competition
- `GET /api/competitions/{code}/matchup-dates?teamA=X&teamB=Y`
  - validates both teams against that competition
  - returns real scheduled events with resolved home/away orientation

### .NET: `DevCore.Api/Sports/CompetitionCatalog.cs`

this is the current platform registry for competition-aware routing.

it defines:
- supported competition codes: `nfl`, `ncaaf`, `nba`, `ncaamb`, `mlb`
- user-facing labels for sport family and level
- odds api key mapping where schedule lookup is supported
- `MaxGroundedSignals` per competition for confidence calibration
- visible-but-unsupported college baseball metadata

this is intentionally thin. it is not a giant league framework.

### .NET: `DevCore.Api/Sports/OddsScheduleClient.cs`

typed `HttpClient` that looks up scheduled matchup events from the odds api.

- `GetEventsAsync(competition, teamA, teamB)`
- competition code maps to odds api key via `CompetitionCatalog`
- supported schedule keys:
  - `nfl` → `americanfootball_nfl`
  - `ncaaf` → `americanfootball_ncaaf`
  - `nba` → `basketball_nba`
  - `ncaamb` → `basketball_ncaab`
  - `mlb` → `baseball_mlb`
- cache is still non-directional by team pair
- provider orientation is still preserved in `MatchupEventDto`
- light fuzzy team matching was added to reduce college-name drift between the reference table and the odds provider

### .NET: `DevCore.Api/Sports/EspnBasketballScheduleClient.cs`

typed `HttpClient` that derives a thin rest/schedule signal for `nba` and `ncaamb` from public espn scoreboard data.

- `GetRestContextAsync(competition, homeTeam, awayTeam, gameDate)`
- source coverage:
  - `nba` → `site.api.espn.com/apis/site/v2/sports/basketball/nba/scoreboard`
  - `ncaamb` → `site.api.espn.com/apis/site/v2/sports/basketball/mens-college-basketball/scoreboard?groups=50&limit=500`
- looks back up to 14 days before the target game date
- finds each team's most recent prior game date
- computes:
  - `HomeLastGameDate`
  - `HomeDaysRest`
  - `HomeIsBackToBack`
  - `AwayLastGameDate`
  - `AwayDaysRest`
  - `AwayIsBackToBack`
- returns `null` unless both teams can be grounded from real schedule data
- cache is per competition + date for 30 minutes to avoid repeated scoreboard fetches
- fuzzy name matching is intentionally conservative and exists only to bridge odds-provider team names to espn scoreboard names

### .NET: `DevCore.Api/Sports/OddsMarketClient.cs`

typed `HttpClient` that retrieves the thinnest possible market signal: current spread only.

- football path still supports:
  - `GetFootballSpreadAsync(competition, homeTeam, awayTeam, gameDate)` for `nfl` and `ncaaf`
- basketball path now supports:
  - `GetBasketballSpreadAsync(competition, homeTeam, awayTeam, gameDate)` for `nba` and `ncaamb`
- both paths use the same odds api provider and competition key mapping from `CompetitionCatalog`
- retrieval stays intentionally thin:
  - current spread only
  - preferred bookmaker selection from the existing bookmaker priority list
  - 15-minute cache
- no line movement history is retrieved here
- because the analyzer payload uses provider-resolved home/away names from schedule lookup, spread matching stays deterministic without a broader team-resolution framework

### .NET: `DevCore.Api/AgentRuns/AgentRunService.cs`

- `ExecuteAsync` still dispatches by `RunType`
- `ExecuteSportsMatchupAsync` now receives `CompetitionMatchupInput`
- retrieve, analyze, and evaluate all key off `competition`, not `sport`
- retriever routing is still intentionally thin:
  - `nfl`, `ncaaf` → fetch market spread
  - `mlb` → fetch probable starters
  - `nba`, `ncaamb` → fetch rest/schedule grounding and current spread
  - all other supported competition-specific sources remain deferred
- confidence calibration now uses `CompetitionMaxGroundedSignals(competition)` plus current grounded count to distinguish zero-grounded, partially grounded, and fully grounded runs. today that mainly changes basketball: `0 of 2`, `1 of 2`, and `2 of 2` no longer calibrate the same.

### .NET: `DevCore.AiClient/SportAnalysisContracts.cs`

request to FastAPI:

```text
SportsAnalysisRequest {
  Competition,
  HomeTeam,
  AwayTeam,
  GameDate,
  FootballMarketContext?,
  MlbStarterContext?,
  BasketballScheduleContext?,
  BasketballMarketContext?
}
```

response from FastAPI:

```text
SportsAnalysisResponse { Lean?, Summary, Confidence, Factors[] }
```

### FastAPI: `services/agent-service/`

- `app/routes/sports.py`
  - validates competition against `{"nfl", "ncaaf", "nba", "ncaamb", "mlb"}`
  - logs `competition=...`
  - dispatches by sport family:
    - football analyzer for `nfl` and `ncaaf`
    - basketball analyzer for `nba` and `ncaamb`
    - mlb analyzer for `mlb`
- `app/services/sports_analyzer.py`
  - prompts are now family-aware instead of pro-only where possible
  - college football and college basketball reuse the football and basketball analyzer families
  - basketball uses an explicit `[schedule data]` block when real rest context is present
  - basketball uses an explicit `[market data]` block when real spread context is present
  - basketball uses explicit no-data fallbacks when schedule or market grounding is unavailable
  - football gets market context when available
  - mlb still gets starting-pitcher context when available

---

## SQL reference data

two platform-level tables still back the reference layer:

**Sports** — now effectively a competition table in current usage.

active rows:
- `nfl`
- `nba`
- `mlb`
- `ncaaf`
- `ncaamb`

**Teams** — active team rows per competition.

current seeded counts:
- NFL: 32
- NBA: 30
- MLB: 30
- NCAAF: 138
- NCAAMB: 365
- total active teams: 595

college baseball is not seeded. it is only represented in the visible competition matrix as unsupported product inventory.

---

## what is stored after a run

`AgentRun` row in SQL Server:

| field | value after a successful sports run |
|---|---|
| RunType | `"sports.matchup.analysis"` |
| Status | `"completed"` |
| InputJson | `{"competition":"ncaaf","homeTeam":"...","awayTeam":"...","gameDate":"..."}` |
| OutputJson | includes `lean`, `summary`, calibrated `confidence`, `factors`, `groundedSignals`, `analyzerConfidence`, `publishability`, `degradationNotes`, `pipelineSteps` |
| Competition | denormalized from input, e.g. `"ncaaf"` |
| GameDate | denormalized from input as `date` column |
| Outcome | null until the learning loop back-fills it |
| AgentProfileKey | null |
| CorrelationId | ASP.NET trace identifier |
| DurationMs | server-side elapsed time |

---

## local dev workflow

scripts live under `scripts/dev/sports/`:

| script | purpose |
|---|---|
| `start-sports-dev.ps1` | launches agent service, platform api, and sports app in separate terminals |
| `stop-sports-dev.ps1` | stops the three dev processes by port |
| `test-sports-dev.ps1` | smoke checks: health ping, nfl analysis, stub detection, unsupported competition gate, full .net chain |

the smoke scripts now use `competition` in the request payloads.

---

## product rule: outward-facing text stays display-safe

internal routing values stay lowercase:
- `nfl`
- `ncaaf`
- `nba`
- `ncaamb`
- `mlb`

outward-facing UI text uses display labels:
- sport family chip uses `Football`, `Basketball`, `Baseball`
- level chip uses `Pro`, `College`
- team names and competition display names come from reference data

the UI should not leak raw internal codes unless there is a true fallback case.

---

## current limitations

| limitation | location | impact |
|---|---|---|
| no auth on sports path | `AgentRunsController`, `SportsApiService` | any caller can create runs; no tenant scoping in production |
| frontend stub path | `SportsApiService`, `environment.development.ts` | demo response remains available for explicit local stub mode; production now calls the real API |
| OutputJson has no schema | `AgentRun.OutputJson` | stored content cannot be queried structurally |
| football market grounding is intentionally thin | `OddsMarketClient`, `sports_analyzer.py` | only current spread is grounded; no line movement history, injury feed, or broader football evidence set yet |
| basketball rest signal is intentionally thin | `EspnBasketballScheduleClient`, `sports_analyzer.py` | grounds back-to-back / days-rest claims only; no travel, injury, or broader schedule-load intelligence yet |
| basketball market grounding is intentionally thin | `OddsMarketClient`, `sports_analyzer.py` | grounds current spread only; no totals, line movement history, or availability feed |
| no reliable shared season-long basketball injury source | retriever layer (deferred) | nba has official injury reporting, but ncaamb still lacks a season-wide shared availability feed; do not force cross-competition injury grounding until a real source exists |
| no college baseball support | `CompetitionCatalog`, frontend level picker | user can see the concept but cannot select a working competition |
| espn schedule source is public but unofficial | `EspnBasketballScheduleClient` | shape can drift over time; retriever must fail closed to the no-data prompt path |

---

## what is not in this flow

these concepts appear in vault docs but are not implemented in this path today:

- competition-specific retriever packages for college sports
- competition-specific evaluator rules beyond `MaxGroundedSignals`
- competition-specific prompt families beyond football / basketball / mlb
- shared basketball injury/availability grounding across `nba` and `ncaamb`
- college baseball support
- outcome tracking or calibration tuning from real results
- delivery, scheduling, billing, or tier enforcement
