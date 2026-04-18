# current handoff: sports matchup analyzer

**date:** 2026-04-18
**status:** dark mode redesign completed. architecture audit completed. RunType dispatch slice completed. strategic vault update completed. run metadata visibility slice completed. structured lean slice completed. output-quality evaluation gate completed. NFL market enrichment slice completed. orchestrator-ready foundation slice completed — ComposeDecisionArtifact seam named; groundedSignals evidence quality contract added; confidence ownership split documented; agent/tool doctrine written.

---

## what was completed in this session

### 9. orchestrator-ready foundation slice

principal-level architecture pass. establishes the seam, confidence ownership split, and evidence quality contract without overbuilding.

**architecture judgment delivered:**
- confidence today is entirely analyzer-local: the model emits it from its training priors against whatever prompt context was provided; no signal scoring, no cross-source weighting
- the analyzer is a synthesizer and collector rolled into one call; there is no separation of concern yet
- final confidence must belong to the orchestrator — when signal scoring exists, `ComposeDecisionArtifact` computes it from evaluator output, not from the analyzer directly

**what was implemented:**

**`groundedSignals` evidence quality field:**
- `SportsAnalysisResponse` (Python + .NET) gains `groundedSignals: list[str] / string[]?`
- `analyze_nfl` sets `["market"]` when `market_context` is present (real spread was retrieved), `[]` otherwise
- `analyze_nba` and `analyze_mlb` always return `[]` (no grounded signals yet)
- `AgentRunExecutionResult` carries `GroundedSignals` into `OutputJson` (stored for the learning loop)
- not surfaced to the UI yet — the orchestrator/evaluator uses it for confidence calibration

**`ComposeDecisionArtifact` seam:**
- extracted from inline `return new AgentRunExecutionResult(...)` in `AgentRunService`
- named private static method with explicit architecture comment
- today: trivial 1:1 mapping. future: receives `EvaluatorOutput` as additional input and computes final lean and confidence from scored signals
- sports-agnostic: the comment explicitly prohibits sport-specific logic here

**contract comments:**
- `SportsAnalysisResponse.Confidence` labeled as "analyzer-local confidence: provisional estimate from this model call"
- `AgentRunExecutionResult.Confidence` labeled as "final confidence (today: passes through from analyzer)"
- `AgentRunExecutionResult` labeled as "current proxy for the decision artifact"
- `_JSON_SHAPE` confidence instruction labeled as "analyzer's local estimate"

**latent serialization bug fixed:**
- `SportsAnalysisRequest.NflMarket` → renamed to `NflMarketContext` so .NET camelCase serialization produces `nflMarketContext` which matches the Python Pydantic field name
- bug was dormant: NFL offseason means the field was always null; would have silently dropped market context during season

**what was NOT built:**
- no static confidence-counting rule
- no multi-agent infrastructure
- no new orchestrator coordinator class
- no new external API
- no UI changes

**files changed:**
- `DevCore.AiClient/SportAnalysisContracts.cs` — `NflMarket` → `NflMarketContext` (serialization fix); `GroundedSignals?` added; architecture comments
- `DevCore.Api/AgentRuns/AgentRunContracts.cs` — `GroundedSignals?` added to `AgentRunExecutionResult`; architecture comments
- `DevCore.Api/AgentRuns/AgentRunService.cs` — `NflMarket:` → `NflMarketContext:` at call site; `ComposeDecisionArtifact` extracted
- `app/models/sports.py` — `groundedSignals: list[str] = []` added to `SportsAnalysisResponse`; `confidence` comment
- `app/services/sports_analyzer.py` — `analyze_nfl` sets `groundedSignals`; `_parse_response` comment; `_JSON_SHAPE` confidence comment

**vault docs changed:**
- `orchestration.md` — seam location mapped to code; agent/tool doctrine written; confidence ownership split documented; transition path updated
- `decision-intelligence-model.md` — section 4 updated: `AgentRunExecutionResult` labeled as proxy, confidence ownership clarified

---

### 8. nfl market enrichment slice

added real nfl spread data from the odds api to the analysis prompt. eliminates the fabricated "line has moved toward X" claims that were the primary quality problem identified in the evaluation gate.

**what changed:**

- `OddsMarketClient` (new, `DevCore.Api/Sports/`) — typed `HttpClient` that calls `/v4/sports/americanfootball_nfl/odds?regions=us&markets=spreads` bracketed to the game date. caches results 15 minutes. returns `NflMarketContext?` — null on any failure (offseason, api error, name mismatch).
- `NflMarketContext` (new, `DevCore.AiClient/`) — record carrying `HomeTeam`, `AwayTeam`, `HomeSpread`, `AwaySpread`, `Bookmaker`, `UpdatedAt`. defined in the ai client layer because it is part of the .NET→FastAPI contract.
- `SportsAnalysisRequest` extended — new optional `NflMarket: NflMarketContext? = null` field. existing call sites unaffected (named params).
- `AgentRunService` — for NFL runs: calls `OddsMarketClient.GetNflSpreadAsync()` before building the ai request; attaches result (or null) to `SportsAnalysisRequest`.
- `Program.cs` — `OddsMarketClient` registered with `AddHttpClient<OddsMarketClient>` pointing to `api.the-odds-api.com`.
- `app/models/sports.py` — `NflMarketContext` Pydantic model added; `SportsAnalysisRequest` extended with `nflMarketContext: NflMarketContext | None = None`.
- `app/services/sports_analyzer.py`:
  - `analyze_nfl` now accepts `market_context: NflMarketContext | None`
  - when present: injects a `[market data]` block into the user message (home spread, away spread, bookmaker, timestamp)
  - when absent: injects `[no market data available]` and explicitly instructs the model not to fabricate spread or line direction
  - `_NFL_SYSTEM` updated: "line movement: note direction, magnitude, and timing" → "market: use only if `[market data]` was provided. do not fabricate."
  - `_JSON_SHAPE` example updated: "line movement since open" → "divisional home record" (removes market-implying example)
- `app/routes/sports.py` — passes `req.nflMarketContext` through to `analyze_nfl`.

**what was deferred (explicit):**
- line movement (opening vs current) — not available via standard `/odds` endpoint; requires historical snapshot endpoint (premium). current spread only.
- nba/mlb market enrichment — nfl only for this slice.
- sharp/public split — removed from nfl system prompt to reduce fabrication surface; no integration.

**verified behavior (live tests):**
- nfl offseason (april 2026): `OddsMarketClient` returns null, model gets "no market data" instruction, outputs `signals are split` with no line/spread claims. confidence 0.5–0.65.
- nfl with injected spread (direct fastapi test): model cites actual spread in the `market` factor without inventing movement direction. lean uses market positioning as a grounded reason.
- nba regression: completely unaffected — `analyze_nba` signature unchanged, no market context sent.
- mlb regression: completely unaffected.

**ownership note:** market data retrieval lives in .NET (not FastAPI) because the api key is in .net user secrets and deterministic retrieval belongs in the platform layer. fastapi remains a pure ai reasoning layer that uses structured context from the request, not raw api calls.

**files changed:**
- `DevCore.AiClient/NflMarketContext.cs` (new)
- `DevCore.AiClient/SportAnalysisContracts.cs` — `NflMarket?` added to `SportsAnalysisRequest`
- `DevCore.Api/Sports/OddsMarketClient.cs` (new)
- `DevCore.Api/Program.cs` — `OddsMarketClient` registered
- `DevCore.Api/AgentRuns/AgentRunService.cs` — `OddsMarketClient` injected; NFL spread lookup added
- `dai/services/agent-service/app/models/sports.py` — `NflMarketContext` + `nflMarketContext` field added
- `dai/services/agent-service/app/services/sports_analyzer.py` — `analyze_nfl` updated; `_NFL_SYSTEM` updated; `_JSON_SHAPE` example fixed
- `dai/services/agent-service/app/routes/sports.py` — market context passed to `analyze_nfl`
- `dai/services/agent-service/.env.example` — note added that odds api key stays in .net

---

### 7. output-quality evaluation gate

verified the full live HTTP path and ran 7 grounded evaluation samples (NFL x3, NBA x2, MLB x2) through the real stack.

**live-path verification:**
- fresh uvicorn started on port 8001 — confirmed returning `lean` in HTTP responses
- .NET API restarted with `ASPNETCORE_ENVIRONMENT=Development AiService__BaseUrl=http://127.0.0.1:8001`
- full POST to `/api/agent-runs` confirmed `lean` present in end-to-end HTTP response
- root cause of stale service: uvicorn `--reload` on port 8000 did not reload after Python edits; Windows session isolation blocked `Stop-Process` from a different terminal session. prevention: always stop via `stop-sports-dev.ps1` from the owning session before editing Python files.

**evaluation rubric: lean coherence, team-name usage, lean/summary distinction, factor usefulness, confidence realism, fabrication risk, sport appropriateness**

**findings:**

| # | sport | matchup | lean | confidence | fabrication risk |
|---|---|---|---|---|---|
| 1 | NBA | Celtics vs Heat | signals are split | 0.70 | low — structural |
| 2 | NFL | Chiefs vs Bills | edge toward Chiefs (line mvmt) | 0.80 | high — invented line direction |
| 3 | NFL | Eagles vs Cowboys | edge toward Eagles (line mvmt) | 0.80 | high — invented line direction |
| 4 | MLB | Dodgers vs Giants | edge toward Dodgers (ERA) | 0.80 | high — invented ERA comparison |
| 5 | MLB | Yankees vs Red Sox | signals are split | 0.70 | medium — bullpen workload invented |
| 6 | NBA | Nuggets vs Lakers | edge toward Nuggets (home/rest) | 0.80 | low — structural |
| 7 | NFL | Titans vs Colts | signals are split | 0.50 | medium — injuries labeled "questionable" |

**what is working:**
- lean coherence: all 7 pass — direction always consistent with summary and factors
- team-name usage: all 7 pass — actual team names used in every directional lean
- lean/summary distinction: adequate — lean names direction + top reason in one sentence; summary adds context without repeating verbatim
- sport vocabulary: correct throughout — NFL (line, situational), MLB (ERA, bullpen, lineup), NBA (pace, rest, home record)
- "signals are split" calibration: appears appropriately for balanced or zero-signal cases

**critical problem:**
fabrication. the model presents invented specifics as real data — line movement directions, ERA comparisons, bullpen workload statuses, injury states. NFL directional cases are the worst: "line has moved toward Chiefs since opening" reads like a real signal but is entirely made-up. MLB pitching comparisons invent specific ERA advantage with no real starter data. NBA structural cases are the best because home court and rest are general structural facts rather than specific fabricated values.

**confidence distribution:**
stepped rather than continuous. directional cases: uniformly 0.80. split (medium signal): uniformly 0.70. split (zero signal): 0.50. model cannot express calibrated confidence because it has no real signal strength to measure against.

**decision:** option A — real signal inputs. fabrication is the root-cause quality problem. confidence calibration on top of fabricated signals still produces fabricated output. real line data gives the model something to anchor lean and confidence against rather than inventing it.

---

### 6. structured lean slice

threaded lean end-to-end from the FastAPI prompt through all .NET contracts to the Angular UI rendering layer. no Angular template changes were needed — the `currentLean()` computed and `Current Lean` module were already wired and waiting.

**what lean is:**
- one sentence naming which side the signal picture currently favors and the single strongest reason
- format: `"edge toward [team name] based on [primary reason]"` or `"signals are split"` when evidence is balanced
- not a pick, not a prediction, not a score. a directional signal using the model's training context.
- listed before summary in the response contract to reflect decision-first ordering

**the prompt instruction (in `_JSON_SHAPE`, shared across all three sport prompts):**
- instructs the model to use actual team names (not "home team" / "away team")
- directs `"signals are split"` when evidence is balanced
- forbids score, spread, and bet predictions
- example provided: `"edge toward Chiefs based on rest advantage and line movement since open."`

**contract chain:**
- FastAPI `SportsAnalysisResponse.lean: str | None` → .NET `SportsAnalysisResponse.Lean: string?` → `AgentRunExecutionResult.Lean: string?` → `AgentRunResultDto.Lean: string?` → Angular `AgentRunResultDto.lean?: string | null` (unchanged)

**null handling:** lean is optional throughout. the Pydantic model defaults to `None`. `_parse_response` normalizes empty or non-string values to `None`. all downstream records use `string? / null`. the Angular `currentLean()` computed already handles null by showing the "Lean unavailable" fallback.

**files changed:**
- `dai/services/agent-service/app/models/sports.py` — `lean: str | None = None` added to `SportsAnalysisResponse`
- `dai/services/agent-service/app/services/sports_analyzer.py` — `_JSON_SHAPE` updated with lean instruction; `_parse_response` updated to extract and normalize lean
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs` — `string? Lean` added to `SportsAnalysisResponse`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs` — `string? Lean` added to both `AgentRunExecutionResult` and `AgentRunResultDto`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunService.cs` — `Lean: aiRes.Lean` threaded into `AgentRunExecutionResult`
- `dai/platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs` — `execution.Lean` threaded into `AgentRunResultDto` constructor
- `dai/apps/sports-app/src/app/sports-api.service.ts` — lean added to stub response; stub factors updated to match the real format (`"category: observation"`)

**what the UI renders:** the `Current Lean` module in `Matchup Read` now shows the lean text as the primary value (large, `text-[#eef4fb]`) with "Structured lean from the current read." as the supporting note. no template changes were required.

---

### 5. run metadata visibility slice

surfaced server-side run metadata in the Analysis Details block of the control rail. no backend changes were required — all data was already in the response DTO.

**what is now visible:**
- **Run time** — server-reported `durationMs` formatted as human-readable `Xs` or `Xms`. conditional: only shown when the backend provides a non-null value (hidden in future stub-only or legacy response cases).
- **Run ID** — `agentRunId` displayed as first 8 characters in monospace with the full UUID available on hover via `title` attribute. always shown after a completed run.

**placement:** Analysis Details block in the left control rail, below Signal quality. follows the exact same label: value row pattern already established. Run ID uses `font-mono text-[#9eb0c8]` (secondary, muted) to signal it is a diagnostic identifier rather than a primary value.

**files changed:**
- `dai/apps/sports-app/src/app/app.ts` — `analysisDetails` computed extended to emit `duration: string | null` and `runId: string`; `formatDuration(ms)` method added
- `dai/apps/sports-app/src/app/app.html` — two new rows added inside the Analysis Details block
- `dai/apps/sports-app/src/app/sports-api.service.ts` — `durationMs: 500` added to stub response for dev consistency

**what is still hidden / not surfaced:**
- `RunType` — stored in DB, not in the result DTO, not surfaced
- `CorrelationId` — stored in DB only; the full UUID on hover is sufficient for cross-referencing FastAPI logs via `X-Agent-Run-Id`
- `createdUtc` — in the DTO, not surfaced; adds noise without diagnostic value
- `GET /api/agent-runs/{id}` — endpoint works, UI still never calls it; wiring it up would require a new type, service method, and result display — deferred

**cross-layer correlation path (unchanged):** `agentRunId` in the UI → `AgentRun.AgentRunId` in DB → `X-Agent-Run-Id` header in FastAPI logs. the Run ID row in the UI makes this correlation path usable without opening DevTools or the DB.

---

### 4. dark mode redesign + final ui polish

complete visual redesign from the light warm editorial direction to a dark navy desktop product interface. all ui files updated. several ui chrome elements removed. hover system extended.

**visual direction change:**
- old direction (2026-04-16, superseded): warm off-white cards (`#f6f3ee`) on a cool-steel diagonal gradient field
- new direction: near-black base field with four-tier dark navy card surfaces and cobalt accent system
- visual language influence: premium desktop product software (Docker Desktop family — layered dark surfaces, restrained accent, enterprise tone)

**page background:**
- `body::before` rewritten to `linear-gradient(150deg, #04080e 0%, #060e18 40%, #091623 100%)` — clean, no radial layers
- previous atmospheric radial removed — its center at `50% -5%` was positioned near peak intensity at viewport top, producing a visible blue glow band
- html fallback: `#060e18`

**card surface system (three CSS classes added to `styles.css`):**
- `.card-surface`: `linear-gradient(160deg, #142238 0%, #0f1b2e 55%, #0c1829 100%)` — hero, control rail, result card
- `.card-surface-deep`: `linear-gradient(160deg, #0f1b2e 0%, #0c1829 55%, #091422 100%)` — factor breakdown section
- inset `#0c1628`: stat cells, empty-state containers; `inset 0 2px 4px rgba(0,0,0,0.25)` recessed shadow
- all main cards: `ring-1 ring-[#2a3f5a]/80` + `inset 0 1px 0 rgba(255,255,255,0.06)` top edge highlight

**ui chrome removed:**
- top cobalt stripe (`border-t-4 border-[#2b74ff]` on page wrapper) — was the visible blue line at the very top of the page
- `PRIMARY READ` eyebrow pill above Matchup Read
- `CONTRIBUTING FACTORS` eyebrow pill above Factor Breakdown
- helper text sentence under Factor Breakdown

**unified divider system:**
- `border-b border-[#22344a] pb-4 lg:pb-5` wrapping the heading block in all four main section headers: hero, configure matchup, matchup read, factor breakdown
- exact same class string used in all four locations — one consistent system

**hover system:**
- `.premium-surface` (existing class): factor cards and lean card — border `#29415c`, bg `#16263c`, drop shadow
- `.inset-surface` (new class): stat cells (Status, Confidence) — border `#29415c`, bg `#111e2e`; restrained lift from deep inset base; both classes share identical `210ms var(--premium-ease)` transition timing

**files changed in this session:**
- `dai/apps/sports-app/src/styles.css` — full token replacement; `.card-surface`, `.card-surface-deep`, `.inset-surface` added; page background rewritten; `html` fallback updated
- `dai/apps/sports-app/src/app/app.html` — all inline color classes replaced; chrome removed; divider system added; hover classes applied
- `dai/apps/sports-app/src/app/team-picker/team-picker.component.html` — all inline color classes replaced with dark equivalents
- `dai/apps/sports-app/src/app/app.ts` — `resultTone()` and `narrationClass()` updated to dark computed values

---

### previously completed (2026-04-16)

three workstreams closed out in the prior session:

### 1. ui / visual polish slice

the matchup analyzer visual direction is now stable and should not be casually reopened. what was stabilized:

- **page background**: diagonal `135deg` gradient, darker cool-steel at top-left (`#8a9aae`), opening to lighter airy tone at bottom-right (`#dce5ed`), with a soft support radial reinforcing the lighter corner. fallback `#b8c4cd`.
- **gradient direction is locked**: darker top-left, lighter bottom-right. do not revert to a vertical gradient or flip the axis.
- **page architecture**: hero shell → left control rail + right `Matchup Read` card → full-width `Factor Breakdown`. single-page scroll. no tabs. stable.
- **warm/cool contrast**: warm off-white cards (`#f6f3ee`) on a cool atmospheric field. this is the identity.
- **premium-control shadow** bumped to `0 1px 3px rgba(10,18,36,0.10), 0 4px 14px rgba(10,18,36,0.07)`.

### 3. RunType dispatch + request-contract clarity

`AgentRunService` no longer hardcodes sports analysis. dispatch is now explicit.

**what changed:**
- `RunTypes` static class added to `AgentRunContracts.cs` — the single registry of known run type strings
- `AgentRunService.ExecuteAsync` dispatches by `req.RunType` using a switch expression
- `"sports.matchup.analysis"` routes to `ExecuteSportsMatchupAsync` (private method)
- unknown run types throw `NotSupportedException` — controller catches, marks run `failed`, persists `ErrorMessage`
- `CreateAgentRunRequest.Input` retains its `SportsMatchupInput` type with a clear comment documenting the limitation and the generalization path when a second run type arrives
- sports flow behavior is unchanged

**files changed:**
- `DevCore.Api/AgentRuns/AgentRunService.cs` — dispatch switch + extracted `ExecuteSportsMatchupAsync`
- `DevCore.Api/AgentRuns/AgentRunContracts.cs` — `RunTypes` class added; `SportsMatchupInput` comment added

---

### 2. architecture audit + correlation plumbing

full read of actual code vs. vault docs. one stale claim corrected. one real gap addressed.

**what was corrected:**
- `current-agent-run-contract.md` claimed `CorrelationId` was "never set by any caller" — wrong. the controller sets it to `HttpContext.TraceIdentifier`. corrected.

**what was built:**
- `X-Agent-Run-Id` header now forwarded from .NET to FastAPI on every sports analyze call
- FastAPI logs `agent_run_id` on every request and error
- `AgentRunResultDto` now returns `durationMs` (server-side execution time)
- `IAgentRunService.ExecuteAsync` now accepts `Guid agentRunId`

**correlation truth documented:**
- `AgentRun.CorrelationId` = `HttpContext.TraceIdentifier` (ASP.NET short trace id, stored in DB)
- `X-Correlation-Id` sent to FastAPI = `Activity.Current?.Id` (W3C trace format — different value, FastAPI logs only)
- `X-Agent-Run-Id` = `AgentRun.AgentRunId` GUID — the stable cross-layer anchor. use this to link FastAPI logs to DB rows.

---

## current implemented truth

### end-to-end flow (working)

```
Angular sports-app
  → GET /api/sports
  → GET /api/sports/{slug}/teams
  → GET /api/sports/{slug}/matchup-dates?teamA=X&teamB=Y
      → OddsScheduleClient.GetEventsAsync() (non-directional, cached 30min)
      ← MatchupEventDto[] { date, homeTeam, awayTeam }
  [user selects date pill → selectedEvent resolved]
  → POST /api/agent-runs
    → AgentRunsController.Create()
      → IdentityResolver (dev bypass: TenantKey=1, UserKey=1)
      → INSERT AgentRun (status=pending, CorrelationId=TraceIdentifier)
      → AgentRunService.ExecuteAsync(req, run.AgentRunId, ct)
        → FastApiClient.AnalyzeSportsMatchupAsync(req, agentRunId, ct)
          → POST http://127.0.0.1:8001/api/sports/analyze  ← port 8001 (fresh); port 8000 is a stale process that cannot be killed from a different terminal session
              X-Correlation-Id: Activity.Current?.Id
              X-Agent-Run-Id: AgentRun.AgentRunId   ← cross-layer anchor
            → sports.py: validates sport, logs agent_run_id + correlation_id
            → sports_analyzer.py: gpt-4o-mini, sport-specific prompt, temperature=0.3
          ← SportsAnalysisResponse { lean, summary, confidence, factors }
      → UPDATE AgentRun (completed, OutputJson, DurationMs)
      ← AgentRunResultDto { agentRunId, status, lean, summary, confidence, factors, createdUtc, durationMs }
  ← render: hero / control rail / Matchup Read / Factor Breakdown
```

### what is truly working today

- SQL reference data (Sports, Teams tables)
- OddsAPI schedule lookup with non-directional cache
- **NFL market enrichment:** `OddsMarketClient` fetches current spread before each NFL analysis; result injected into the user message as a `[market data]` block; null result triggers no-market fallback that explicitly prevents fabrication
- AgentRun persistence: pending → completed/failed with InputJson, OutputJson, DurationMs, CorrelationId
- FastAPI sports analysis: sport-specific prompts, gpt-4o-mini, structured JSON response with `lean`, `summary`, `confidence`, `factors[]`
- `lean` field: one-sentence directional signal, propagated through all contract layers, rendered in the `Current Lean` module
- `X-Agent-Run-Id` forwarded on every sports call; FastAPI logs it
- `durationMs` and `agentRunId` surfaced in the Analysis Details block of the UI
- `AgentRunService` dispatches by `RunType`; `"sports.matchup.analysis"` routes correctly; unknown types fail loudly with a stored error

### what is scaffolded / not yet active

- `AgentProfileKey` — stored null, no profile loaded at runtime
- `CreateAgentRunRequest.Input` typed as `SportsMatchupInput` — will need generalization for a second run type
- `GET /api/agent-runs/{id}` — endpoint exists, works, UI never calls it
- all collector/evaluator/synthesizer/signal-scoring — in vault docs only, no code

---

## what docs were corrected

| doc | what changed |
|---|---|
| `current-agent-run-contract.md` | CorrelationId claim corrected; cross-layer correlation table added; `durationMs` added to response shape; `AgentRunService` hardcoding gap documented |
| `current-sports-analysis-flow.md` | X-Agent-Run-Id header added to flow map and layer descriptions; `durationMs` added to DTO reference; FastAPI log line updated |
| `ui-concept.md` | background system updated from stale `#ced6de` / vertical radial description to current diagonal system with exact token values; gradient direction rule added; visual direction marked stable; `durationMs` surfacing added to near-term refinements |
| this handoff | complete rewrite to reflect completed slices and clear next-step |
| `ui-concept.md` | visual system section replaced with dark token system and four-tier card hierarchy; interaction language updated with `.inset-surface`; hierarchy decisions block updated with eyebrow removal and divider system; visual direction note updated to locked; deferred table dark mode toggle corrected |
| this handoff | dark mode redesign session added as section 4; decisions already made updated with dark direction decisions; remaining open (ui) updated with touch hover validation |

---

## architecture gaps (current priority order)

1. `CreateAgentRunRequest.Input` typed as `SportsMatchupInput` — sports-specific; will need to become a generic envelope before a second run type can be added. documented with a comment in the record.
2. `OutputJson` has no schema — stored as raw JSON blob; cannot be queried or validated structurally
3. no auth on `AgentRunsController` — dev bypass only; not production-safe
4. `useStubApi = true` in `environment.ts` production build — real API only called in dev
5. `GET /api/agent-runs/{id}` — endpoint exists and works; UI still never calls it; wiring it up is the next natural step for making individual runs inspectable
6. no run history in UI — runs accumulate in DB with no visibility surface

---

## next slice candidates

orchestrator-ready foundation is in place. seam named. evidence quality tracked. agent/tool doctrine written.

**recommended next: promote the collect step to a typed `CollectorOutput`.**

this is the clearest aligned move from the `orchestration.md` transition path (step 4). today, market data is retrieved in `ExecuteSportsMatchupAsync` and immediately embedded into `SportsAnalysisRequest` with no intermediate shape. promoting it to a typed `CollectorOutput` means:
- the collect step has an explicit output contract (nullable fields, source statuses, retrieved_at)
- the compose seam receives a richer input
- `GroundedSignals` population moves out of the analyzer and into the collector output
- the architecture matches the target model without introducing a full orchestrator

**scope for this slice (thin):**
- define `SportsCollectorOutput` in `DevCore.Api` (or a new `Collector` layer): `{ NflMarketContext?, RetrievedAt, SourceStatuses }`
- `AgentRunService.ExecuteSportsMatchupAsync` produces a `SportsCollectorOutput` from the collect step
- `ComposeDecisionArtifact` takes `SportsCollectorOutput` as an additional input (today: reads `GroundedSignals` from it)
- `SportsAnalysisRequest` no longer needs to carry `NflMarketContext` as a prompt-injection concern — that stays in FastAPI; the collector output just proves what was retrieved

actually, on reflection: the simplest version of this is to keep the current structure and only formalize the `CollectorOutput` naming and type. the full architectural split (collector as a distinct class with its own interface) should wait until there are at least 2 distinct collection sources per sport.

**alternative next: MLB starting pitcher injection**
mlb fabrication risk is still the second-worst after nfl (model invents ERA comparisons, bullpen workloads). `statsapi.mlb.com` is free and returns probable starter names and recent stats. this follows the same pattern as nfl market enrichment: retrieve before the model call, inject into the prompt, mark `groundedSignals=["starting_pitching"]`. fits in one session.

**do not build:**
- static confidence-counting rule pinned to signal quantity (rejected: wrong seam, wrong level)
- full orchestrator coordinator class (not yet — one pipeline configuration doesn't justify it)
- new UI features
- delivery, scheduling, billing

do not build delivery, scheduling, billing, or collector/evaluator pipelines until the brief itself is worth delivering.

---

## future direction context

the full strategic model is documented in `02 Platform/architecture/decision-intelligence-model.md`. this section is a summary for handoff continuity.

**what this platform is building toward (not what exists today):**

the sports matchup analyzer is the first expression of a decision intelligence system. the long-term product is not a report generator — it is a system that collects real evidence from external sources, scores each signal category explicitly, synthesizes a structured decision artifact with lean + counter-signals + source provenance, and tracks outcomes to calibrate signal quality over time.

**seven architectural directions — all longer-term, none active today:**

| direction | current state | target |
|---|---|---|
| decision-first model | lean is implicit in summary text | structured lean field in response with direction, strength, basis |
| evidence-backed architecture | single LLM call using model training weights | collector layer retrieves real signals; signals become typed prompt inputs |
| orchestrated synthesis | `AgentRunService` calls FastAPI directly, one step | orchestrator coordinates collect → evaluate → synthesize with typed handoffs |
| decision artifact | `{ summary, confidence, factors[] }` | full artifact with lean, signals[], counter_signals[], sources[], provenance |
| learning loop | runs stored; no outcome data; nothing tracked | outcome collector stores results; accuracy tracked per signal category over time |
| storage by purpose | single SQL DB, JSON blobs | relational + object + vector (deferred) + analytics (deferred) |
| profile-driven behavior | `AgentProfileKey` null; prompts hardcoded | versioned agent profiles injecting model, temperature, prompt, schema, signal weights |

**the sequencing rule:** current state → lean → real signals → typed pipeline steps → orchestrator → profiles → learning loop. each layer must prove itself before the next is introduced. do not jump ahead.

---

## decisions already made — do not reopen without new evidence

**ui / visual:**
- tabs vs scroll — single-page scroll is correct
- filler cards for layout symmetry — do not add
- one-off sport control — sport uses the shared picker
- fabricated lean or pick text — neutral fallback only
- visual direction — dark navy workspace, cobalt accent, four-tier card depth hierarchy. locked. do not reintroduce light or warm surfaces.
- page background — `body::before` fixed layer, clean linear gradient only. no radial glow layers.
- eyebrow pill labels — removed. `Matchup Read` and `Factor Breakdown` stand alone as section titles. do not re-add pill labels as section chrome.
- divider system — `border-b border-[#22344a] pb-4 lg:pb-5` under all four main section headers. one class, consistent across the page.
- hover tiers — `.premium-surface` for factor cards and lean card; `.inset-surface` for stat cells. do not apply the same class to both surface types.

**architecture:**
- `X-Agent-Run-Id` as the cross-layer correlation anchor — settled
- `durationMs` in the API response — in place

---

## active implementation files

**Angular:**
- `dai/apps/sports-app/src/app/app.html`
- `dai/apps/sports-app/src/app/app.ts`
- `dai/apps/sports-app/src/app/core/models/agent-run.model.ts`
- `dai/apps/sports-app/src/app/sports-api.service.ts`
- `dai/apps/sports-app/src/app/team-picker/team-picker.component.ts`
- `dai/apps/sports-app/src/app/team-picker/team-picker.component.html`
- `dai/apps/sports-app/src/styles.css`

**.NET:**
- `dai/platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunService.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/IAgentRunService.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs`
- `dai/platform/dotnet/DevCore.AiClient/FastApiClient.cs`
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs`

**FastAPI:**
- `dai/services/agent-service/app/routes/sports.py`

---

## remaining open (ui)

- browser-verify empty, populated, mobile, and long-scroll states in a real browser (in dark direction)
- validate picker behavior across mouse, keyboard, and touch
- validate hover states on touch devices — `.premium-surface` and `.inset-surface` are wrapped in `@media (hover: hover)` and should not apply on touch; confirm no unintended static state change on mobile
- verify hover states on touch devices once tested in a real mobile browser — `.premium-surface` and `.inset-surface` are wrapped in `@media (hover: hover)` and should not apply on touch

## remaining open (platform)

- apply `RenameOaklandToSacramento` migration to running DB instance
- auth on `AgentRunsController` before any production deployment
- `useStubApi` in `environment.ts` — confirm production build value before any real deployment
