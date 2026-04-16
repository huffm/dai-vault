# current handoff: sports matchup analyzer

**date:** 2026-04-16
**status:** ui/visual slice completed. architecture audit completed. next slice identified.

---

## what was completed in this session

two parallel workstreams closed out:

### 1. ui / visual polish slice

the matchup analyzer visual direction is now stable and should not be casually reopened. what was stabilized:

- **page background**: diagonal `135deg` gradient, darker cool-steel at top-left (`#8a9aae`), opening to lighter airy tone at bottom-right (`#dce5ed`), with a soft support radial reinforcing the lighter corner. fallback `#b8c4cd`.
- **gradient direction is locked**: darker top-left, lighter bottom-right. do not revert to a vertical gradient or flip the axis.
- **page architecture**: hero shell → left control rail + right `Matchup Read` card → full-width `Factor Breakdown`. single-page scroll. no tabs. stable.
- **warm/cool contrast**: warm off-white cards (`#f6f3ee`) on a cool atmospheric field. this is the identity.
- **premium-control shadow** bumped to `0 1px 3px rgba(10,18,36,0.10), 0 4px 14px rgba(10,18,36,0.07)`.

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
          → POST http://127.0.0.1:8000/api/sports/analyze
              X-Correlation-Id: Activity.Current?.Id
              X-Agent-Run-Id: AgentRun.AgentRunId   ← cross-layer anchor
            → sports.py: validates sport, logs agent_run_id + correlation_id
            → sports_analyzer.py: gpt-4o-mini, sport-specific prompt, temperature=0.3
          ← SportsAnalysisResponse { summary, confidence, factors }
      → UPDATE AgentRun (completed, OutputJson, DurationMs)
      ← AgentRunResultDto { agentRunId, status, summary, confidence, factors, createdUtc, durationMs }
  ← render: hero / control rail / Matchup Read / Factor Breakdown
```

### what is truly working today

- SQL reference data (Sports, Teams tables)
- OddsAPI schedule lookup with non-directional cache
- AgentRun persistence: pending → completed/failed with InputJson, OutputJson, DurationMs, CorrelationId
- FastAPI sports analysis: sport-specific prompts, gpt-4o-mini, structured JSON response
- `X-Agent-Run-Id` forwarded on every sports call; FastAPI logs it
- `durationMs` returned in API response

### what is scaffolded / not yet active

- `AgentProfileKey` — stored null, no profile loaded at runtime
- `AgentRunService` — hardcodes sports; no dispatch by `RunType`
- `lean` field — reserved in Angular model, backend does not populate
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

---

## architecture gaps (current priority order)

1. **`AgentRunService` hardcodes sports** — no dispatch by `RunType`. any second run type silently calls the sports endpoint. this is the primary structural gap. **this is the next slice.**
2. `OutputJson` has no schema — stored as raw JSON blob; cannot be queried or validated structurally
3. no auth on `AgentRunsController` — dev bypass only; not production-safe
4. `useStubApi = true` in `environment.ts` production build — real API only called in dev
5. `durationMs` is now in the response but the UI still derives signal quality from confidence only — the server-side timing is available to surface
6. no run history in UI — runs accumulate in DB with no visibility surface

---

## next slice: RunType dispatch + request-contract clarity

**this is the correct next move. it is not optional before adding a second run type.**

### why now

- correlation plumbing is clean: `X-Agent-Run-Id` links FastAPI logs to DB rows
- `AgentRunService.ExecuteAsync` has no dispatch logic — it unconditionally calls `FastApiClient.AnalyzeSportsMatchupAsync`. this will silently misfired for any run type that is not `sports.matchup.analysis`
- the service boundary between "what the platform does" and "what the sports analyzer does" is blurry — `AgentRunService` knows about `SportsAnalysisRequest` directly, which it should not
- fixing this before adding more run types is far cheaper than fixing it after

### what the slice involves

refactor `AgentRunService` to:
- dispatch by `req.RunType` rather than calling the sports endpoint unconditionally
- for `"sports.matchup.analysis"`: delegate to a sport-specific handler (can be a private method or a dedicated `SportsMatchupHandler` class)
- for unknown run types: return a structured error rather than silently misrouting
- make the request/contract boundary explicit — `AgentRunService` should not directly construct `SportsAnalysisRequest`

### likely files touched

| file | change |
|---|---|
| `DevCore.Api/AgentRuns/AgentRunService.cs` | add RunType dispatch; extract sports execution to a handler |
| `DevCore.Api/AgentRuns/IAgentRunService.cs` | interface stays the same; implementation changes |
| `DevCore.Api/AgentRuns/AgentRunContracts.cs` | possibly extract a `SportsMatchupRunInput` typed record separate from the raw `SportsMatchupInput` |
| `DevCore.AiClient/FastApiClient.cs` | no structural change expected |
| `DevCore.AiClient/SportAnalysisContracts.cs` | no structural change expected |
| vault: `current-agent-run-contract.md` | update to reflect dispatch model |
| vault: `current-sports-analysis-flow.md` | update service layer description |

### follow-on slices (after RunType dispatch)

1. **run metadata visibility** — surface `durationMs`, `agentRunId`, and run status in the UI's analysis details block. wire up `GET /api/agent-runs/{id}` for post-run inspection. makes the platform observable from the product surface.
2. **structured result enrichment** — add a `lean` field to the backend contract. improve factor quality. better confidence calibration. this is the work that makes the brief worth delivering.

do not build delivery, scheduling, billing, or collector/evaluator pipelines until the brief itself is worth delivering.

---

## decisions already made — do not reopen without new evidence

**ui / visual:**
- tabs vs scroll — single-page scroll is correct
- filler cards for layout symmetry — do not add
- one-off sport control — sport uses the shared picker
- fabricated lean or pick text — neutral fallback only
- gradient direction — darker top-left, lighter bottom-right, 135° diagonal. stable.
- page background on scroll-sized wrapper — stays on `body::before` fixed layer

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

- browser-verify empty, populated, mobile, and long-scroll states in a real browser
- validate picker behavior across mouse, keyboard, and touch
- surface `durationMs` in the UI analysis details (data is in response; UI is still client-derived)
- add structured lean only when the backend contract supports it

## remaining open (platform)

- apply `RenameOaklandToSacramento` migration to running DB instance
- auth on `AgentRunsController` before any production deployment
- `useStubApi` in `environment.ts` — confirm production build value before any real deployment
