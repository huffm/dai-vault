# implementation plan: sports analysis — stub to first-pass real flow

**date:** 2026-04-10
**scope:** NFL and NBA
**starting point:** current real request path in the dai monorepo

---

## current state of the path

the full request chain from Angular to FastAPI is already wired and working. every layer except the FastAPI endpoint produces real behavior.

### what is real today

| layer | file | status |
|---|---|---|
| Angular form | `apps/sports-app/app.ts` | real — submits sport, homeTeam, awayTeam, gameDate |
| Angular service | `apps/sports-app/sports-api.service.ts` | real — calls `POST /api/agent-runs` (no auth token) |
| .NET controller | `DevCore.Api/Controllers/AgentRunsController.cs` | real — creates AgentRun row, calls service, updates status |
| .NET service | `DevCore.Api/AgentRuns/AgentRunService.cs` | real — translates request, calls FastApiClient |
| .NET HTTP client | `DevCore.AiClient/FastApiClient.cs` | real — calls `POST /api/sports/analyze`, deserializes response |
| AgentRun persistence | `DevCore.Data/AppDbContext.cs` | real — row written with InputJson, OutputJson, Status, timing |

### what is stubbed

| layer | file | what is wrong |
|---|---|---|
| FastAPI endpoint | `services/agent-service/main.py` lines 192–204 | hardcoded response — `confidence: 0.64`, three fixed factors, no OpenAI call |
| Python routes | `app/routes/sports.py` | comment placeholder only |
| Python models | `app/models/sports.py` | comment placeholder only |
| Python service | `app/services/sports_analyzer.py` | comment placeholder only |
| sports-app production env | `environment.ts` | `useStubApi: true` — bypasses the real API entirely in production |

### current contracts (do not break these)

**request** (accepted by .NET, forwarded to FastAPI):
```
sport: string         // "nfl" or "nba"
homeTeam: string
awayTeam: string
gameDate: string      // ISO date
```

**response** (returned from FastAPI, written to AgentRun.OutputJson, returned to Angular):
```
summary: string
confidence: float     // 0.0–1.0
factors: List[str]
```

these contracts are mirrored in three places: `.NET AgentRunContracts.cs`, `FastApiClient SportAnalysisContracts.cs`, and `Angular agent-run.model.ts`. any change must be reflected in all three.

---

## the thinnest useful first implementation

replace the hardcoded stub in `main.py` with a real OpenAI call using only the inputs already available: sport, homeTeam, awayTeam, gameDate. no external data sources yet.

this is useful because:
- it produces variable, non-deterministic output — different games return different briefs
- it validates the full round-trip under real AI latency
- it gives something to evaluate for prompt quality before adding data source complexity
- it requires touching only one file (`main.py`) and one new file (`app/services/sports_analyzer.py`)

**what the prompt does at this stage:**
uses model knowledge (team form, matchup context, scheduling) to produce a 2–3 sentence summary, a confidence float, and 3–5 named factors. the output is soft analysis, not signal-scored intelligence — but it is real and variable.

**sport branching at this stage:**
a single conditional on `sport` selects a sport-specific prompt template. NFL prompt emphasizes weekly context, injuries, and spread movement patterns. NBA prompt emphasizes rest, back-to-back scheduling, and nightly line behavior. same output shape for both.

**what this does not do:**
- does not call any external data source
- does not score signals
- does not compute confidence from signal agreement — confidence is a model estimate
- does not enforce tiers or check Stripe
- does not trigger on a schedule

---

## logic split: FastAPI vs .NET

### FastAPI owns
- prompt construction (sport-specific)
- OpenAI call and response parsing
- all AI logic and signal scoring (future)
- external data source calls (future — stays in Python)
- sport-specific branching rules (weather null for NBA, back-to-back logic, etc.)

### .NET owns
- AgentRun record lifecycle (create, update status, record timing)
- request validation and identity resolution
- tier enforcement (future — check Stripe before calling FastAPI)
- routing: which AgentRunService path to invoke based on RunType
- returning the result DTO to Angular

### the boundary
.NET hands off `SportsAnalysisRequest` to FastAPI and gets back `SportsAnalysisResponse`. .NET does not interpret the response — it stores `OutputJson` and returns the DTO fields as-is. all intelligence is in FastAPI.

this boundary is already correct in the current code. do not move it.

---

## recommended milestones

### milestone 1 — real AI call, no data sources
**what changes:**
- `app/services/sports_analyzer.py`: implement `analyze(sport, home, away, date)` — builds sport-specific prompt, calls `gpt-4o-mini`, parses response into `SportsAnalysisResponse`
- `main.py`: replace hardcoded stub with call to `sports_analyzer.analyze()`
- `app/routes/sports.py`: move the route handler out of `main.py` into this file and register it

**output:** same contract shape as today. summary, confidence, factors — but generated by the model for the actual teams and date.

**validation:** run the endpoint for 5 real NFL games and 5 NBA games. read the summaries. check that NFL and NBA outputs feel distinct. confirm no hardcoded values appear.

---

### milestone 2 — structured output aligned to signal categories
**what changes:**
- `app/models/sports.py`: define an internal `SportsAnalysisDetail` model with named fields: `line_movement_note`, `injury_note`, `situational_note`, `weather_note` (NFL only), `sharp_public_note`
- prompt updated to elicit these fields from the model explicitly, then map them to `factors[]` in the response
- `confidence` derived from how many categories the model could address with high certainty (model self-assessment, not computed from data yet)

**output:** factors are now named signal categories, not freeform strings. `summary` synthesizes across them. the response shape is unchanged — `factors[]` contains more consistent labels.

**validation:** check that factor labels are stable across runs for the same game. check that NBA outputs omit weather factor. check that NFL outdoor games include it.

---

### milestone 3 — one live data source: the odds api
**what changes:**
- new file `app/services/odds_client.py`: calls the odds api for current spread, total, and opening line for the given game
- collector step added to `sports_analyzer.py`: fetch odds before building the prompt; inject real spread data into the prompt
- `confidence` still model-estimated, but now grounded in real line data

**output:** summary and factors reference actual current lines, not model priors. the analysis is now data-grounded, not knowledge-grounded.

**validation:** compare the AI's stated spread to the actual odds api value. they should match. confirm the endpoint still returns within the 90-second .NET timeout.

---

### milestone 4 — second data source: sharp/public split
**what changes:**
- new file `app/services/actionnetwork_client.py`: fetches sharp % and public % for the game
- injected into the prompt alongside odds data
- reverse line movement flag computed in Python and passed as a named factor

**output:** brief now reflects actual betting market positioning, not model assumptions about team quality.

**validation:** manually verify sharp/public data matches actionnetwork for 3 games. confirm reverse line movement flag fires correctly.

---

### milestone 5 — structured signal scoring (evaluator step)
**what changes:**
- `sports_analyzer.py` split into two steps: `collect()` and `evaluate()`
- `evaluate()` calls the model with a scoring prompt (not a narrative prompt) to produce +1/0/-1/null per signal category
- platform aggregate computed in Python: sum of non-null scores, confidence from % scored and in agreement
- `synthesize()` added as a third step: takes the scored table and produces the narrative

**output:** `confidence` is now computed from signal agreement, not model self-assessment. `factors[]` maps to actual scored signals.

**note:** this is the step where the vault's 5-agent pipeline starts to appear in the code. do not attempt this milestone until milestone 4 is stable — the pipeline is only worth splitting once there is real data flowing through it.

---

## what to keep simple in v1

- **input shape**: `sport`, `homeTeam`, `awayTeam`, `gameDate` is sufficient. do not add stadium, venue type, or lat/lon to the frontend input — derive or look up in Python.
- **response shape**: keep `summary`, `confidence`, `factors[]`. do not add a `sections[]` structure until the synthesizer is a separate step (milestone 5).
- **sport handling**: a simple `if sport == "nfl"` branch in the prompt builder is fine. no sport registry, no plugin pattern.
- **error handling**: let the `.NET AgentRun` catch failures via `Status = failed` and `ErrorMessage`. do not add retry logic in Python until the path is stable.
- **model**: `gpt-4o-mini` is the right choice for milestones 1–4. do not switch models until quality is the constraint.

---

## what to defer

| item | why defer |
|---|---|
| rotowire injury data (milestone 6+) | adds latency and a paid dependency — validate the odds+sharp flow works first |
| open-meteo weather data | only useful after odds and sharp/public are in place; NFL outdoor venue lookup adds scope |
| collector/evaluator/synthesizer as separate HTTP calls or agents | premature until the single-function version is working and the split is clearly needed |
| AgentProfile lookup at dispatch time | `AgentProfileId` is accepted but not used — leave it unused until permission enforcement is needed |
| Stripe tier check before dispatch | no billing infrastructure exists; add when free/starter/pro tiers are ready |
| scheduled triggers | no scheduler infrastructure; add in platform phase 3 |
| sports-app authentication | no bearer token on `SportsApiService` calls; add when tenant scoping is required |
| `sports-app` production `useStubApi: true` | flip this only after milestone 1 is validated in dev — do not flip it prematurely |
| gRPC path for sports analysis | only the Python side exists; the .NET HTTP path works and is sufficient for v1 |

---

## open questions before starting milestone 1

1. **team name format**: the odds api returns team names in a specific format (e.g. `"Kansas City Chiefs"`). does the Angular form send full team names or abbreviations? these need to match for odds api lookups in milestone 3.
2. **gameDate format**: Angular sends an ISO date string. the odds api uses event IDs, not dates directly — confirm the lookup strategy before milestone 3.
3. **OpenAI key**: `services/agent-service/.env.example` shows `OPENAI_API_KEY=` is empty. confirm the key is set in the local `.env` before testing milestone 1.
4. **timeout**: the .NET `HttpClient` for FastAPI has a 90-second timeout. an OpenAI call for a sports brief should complete well inside that, but confirm under real network conditions before adding data source calls in milestone 3.
