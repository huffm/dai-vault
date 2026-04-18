# current handoff: sports matchup analyzer

**date:** 2026-04-18
**status:** dark mode redesign completed. architecture audit completed. RunType dispatch slice completed. strategic vault update completed — decision intelligence model documented. (light ui/visual slice from 2026-04-16 is superseded by the dark direction.)

---

## what was completed in this session

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
- `AgentRunService` dispatches by `RunType`; `"sports.matchup.analysis"` routes correctly; unknown types fail loudly with a stored error

### what is scaffolded / not yet active

- `AgentProfileKey` — stored null, no profile loaded at runtime
- `CreateAgentRunRequest.Input` typed as `SportsMatchupInput` — will need generalization for a second run type
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
| `ui-concept.md` | visual system section replaced with dark token system and four-tier card hierarchy; interaction language updated with `.inset-surface`; hierarchy decisions block updated with eyebrow removal and divider system; visual direction note updated to locked; deferred table dark mode toggle corrected |
| this handoff | dark mode redesign session added as section 4; decisions already made updated with dark direction decisions; remaining open (ui) updated with touch hover validation |

---

## architecture gaps (current priority order)

1. `CreateAgentRunRequest.Input` typed as `SportsMatchupInput` — sports-specific; will need to become a generic envelope before a second run type can be added. documented with a comment in the record.
2. `OutputJson` has no schema — stored as raw JSON blob; cannot be queried or validated structurally
3. no auth on `AgentRunsController` — dev bypass only; not production-safe
4. `useStubApi = true` in `environment.ts` production build — real API only called in dev
5. `durationMs` is now in the response but the UI still derives signal quality from confidence only — the server-side timing is available to surface
6. no run history in UI — runs accumulate in DB with no visibility surface

---

## next slice: run metadata visibility

RunType dispatch is done. the platform's service boundary is now explicit.

the next highest-value slice is making the platform observable from the product surface:

### what this involves

- surface `durationMs` in the UI analysis details block (data is already in the response; UI derives timing client-side only)
- wire up `GET /api/agent-runs/{id}` for post-run inspection — endpoint exists and works; UI never calls it
- expose `agentRunId` visibly in the dev/diagnostics area so runs can be cross-referenced against logs

this is a thin UI-side slice. no backend changes required unless the detail DTO needs additions.

### follow-on after run metadata visibility

1. **`CreateAgentRunRequest.Input` generalization** — change `Input` from `SportsMatchupInput` to a generic envelope before introducing a second run type. the comment in the record documents the path.
2. **structured result enrichment** — add `lean` to the backend contract. improve factor quality and confidence calibration. this is the work that makes the brief worth delivering.

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
- surface `durationMs` in the UI analysis details (data is in response; UI is still client-derived)
- add structured lean only when the backend contract supports it

## remaining open (platform)

- apply `RenameOaklandToSacramento` migration to running DB instance
- auth on `AgentRunsController` before any production deployment
- `useStubApi` in `environment.ts` — confirm production build value before any real deployment
