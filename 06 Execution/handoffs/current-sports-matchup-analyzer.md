# current handoff: sports matchup analyzer

**date:** 2026-04-20
**status:** dark mode redesign completed. architecture audit completed. RunType dispatch slice completed. strategic vault update completed. run metadata visibility slice completed. structured lean slice completed. output-quality evaluation gate completed. NFL market enrichment slice completed. orchestrator-ready foundation slice completed. MLB starting pitcher injection slice completed. typed SportsCollectorOutput slice completed. typed EvaluatorOutput + calibrated confidence slice completed — three-step pipeline (collect → analyze → evaluate) now fully established; calibrated confidence comes from the evaluator, not the analyzer; both confidence values stored in OutputJson for the learning loop. app shell + nav slice completed — persistent header, three-route nav (Matchup Analyzer / History / Account), lazy-loaded page components; Saved Reads moved inside History as a filter tab. competition-first slice completed — selection flow is now sport family + level → explicit competition code; supported competitions are NFL, NCAAF, NBA, NCAAMB, and MLB; college baseball is visible as unavailable. basketball rest / schedule grounding slice completed — `nba` and `ncaamb` now ground back-to-back and days-rest claims before the model call.

---

## what was completed in this session

### 15. basketball rest / schedule grounding slice

added a thin grounded evidence source for `nba` and `ncaamb` so basketball no longer always runs as zero-grounded.

**what was implemented:**

- **new explicit basketball schedule context:** added `BasketballScheduleContext` to the .NET ↔ FastAPI contract with only the fields this slice needs:
  - `HomeLastGameDate`
  - `HomeDaysRest`
  - `HomeIsBackToBack`
  - `AwayLastGameDate`
  - `AwayDaysRest`
  - `AwayIsBackToBack`
- **deterministic retrieval stays in the platform layer:** new `EspnBasketballScheduleClient` calls public espn scoreboard endpoints before the model call:
  - NBA: `site.api.espn.com/apis/site/v2/sports/basketball/nba/scoreboard`
  - NCAAMB: `site.api.espn.com/apis/site/v2/sports/basketball/mens-college-basketball/scoreboard?groups=50&limit=500`
- **retrieval stays thin and explicit:** the client looks back up to 14 days, finds each team's most recent prior game date, computes days rest, and marks back-to-back only when `daysRest == 0`.
- **collector ownership preserved:** `AgentRunService.CollectAsync` now branches for `nba` and `ncaamb`, retrieves the schedule context before the ai call, and stores it in `SportsCollectorOutput`.
- **grounded signal ownership preserved:** `SportsCollectorOutput` now computes `groundedSignals=["rest_schedule"]` when `BasketballScheduleContext` is non-null. FastAPI does not set this.
- **basketball analyzer prompt is now conditional on real schedule data:** `analyze_basketball` injects a `[schedule data]` block only when grounded data exists. when not, it injects `[no schedule data available for this game]` and explicitly forbids invented rest or schedule claims.
- **confidence seam now matters for basketball:** `CompetitionCatalog.MaxGroundedSignals` for `nba` and `ncaamb` is now `1`, so evaluator dampening is no longer permanent when this collector succeeds.

**what was intentionally NOT built:**

- no ui changes
- no broad basketball schedule platform
- no travel, three-in-four, or workload model
- no injury collector
- no `nfl`, `ncaaf`, or `mlb` behavior changes beyond shared contract wiring

**files changed in this slice:**
- `dai/platform/dotnet/DevCore.AiClient/BasketballScheduleContext.cs` (new)
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunService.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsCollectorOutput.cs`
- `dai/platform/dotnet/DevCore.Api/Program.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/CompetitionCatalog.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/EspnBasketballScheduleClient.cs` (new)
- `dai/services/agent-service/app/models/sports.py`
- `dai/services/agent-service/app/routes/sports.py`
- `dai/services/agent-service/app/services/sports_analyzer.py`

**vault docs changed in this slice:**
- `02 Platform/architecture/current-sports-analysis-flow.md`
- `02 Platform/architecture/orchestration.md`
- this handoff

### 14. competition-first selector and routing slice

made competition a first-class concept across the live matchup analyzer flow without redesigning the page.

**what was implemented:**

- **competition model introduced end-to-end:** internal routing now keys off explicit competition codes, not a generic sport slug. active supported codes: `nfl`, `ncaaf`, `nba`, `ncaamb`, `mlb`.
- **selector flow updated without redesign:** the analyzer now asks for `Sport` then `Level`, then teams, then date, then analyze. this keeps the existing control rail and visual direction intact while making college support explicit.
- **unsupported combination handled honestly:** baseball + college is present in the level selection matrix as unavailable. it is disabled in the picker and surfaced with the note `College baseball is not available yet.`
- **reference api moved from sports to competitions:** frontend now calls:
  - `GET /api/competitions`
  - `GET /api/competitions/{code}/teams`
  - `GET /api/competitions/{code}/matchup-dates`
- **request contracts renamed to competition:** `SportsMatchupInput` became `CompetitionMatchupInput`; .NET → FastAPI request now uses `competition` instead of `sport`.
- **schedule routing generalized cleanly:** `OddsScheduleClient` now resolves odds api keys from `CompetitionCatalog`, which maps `nfl`, `ncaaf`, `nba`, `ncaamb`, `mlb` to the correct provider keys.
- **collector / evaluator seams preserved:** the collect → analyze → evaluate structure is unchanged. only collector routing moved from sport terminology to competition terminology.
- **college teams seeded into platform reference data:** added real NCAAF and NCAAMB seed data to SQL-backed reference tables:
  - NCAAF: 138 teams
  - NCAAMB: 365 teams
- **analyzer prompts generalized by family where appropriate:** football and basketball analyzers now support both pro and college competitions without inventing unsupported grounded data.

**what was intentionally NOT built:**

- no college baseball support
- no new college-specific evidence collectors yet
- no giant league abstraction framework
- no UI redesign beyond the extra selector step and disabled availability state
- no competition-specific evaluator rule engine beyond the existing `CompetitionMaxGroundedSignals` seam

**files changed in this slice:**
- `dai/apps/sports-app/src/app/analyzer/analyzer.component.ts`
- `dai/apps/sports-app/src/app/analyzer/analyzer.component.html`
- `dai/apps/sports-app/src/app/core/models/agent-run.model.ts`
- `dai/apps/sports-app/src/app/sports-api.service.ts`
- `dai/apps/sports-app/src/app/team-picker/team-picker.component.ts`
- `dai/apps/sports-app/src/app/team-picker/team-picker.component.html`
- `dai/platform/dotnet/DevCore.Api/Sports/CompetitionCatalog.cs` (new)
- `dai/platform/dotnet/DevCore.Api/Controllers/SportsReferenceController.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunService.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsCollectorOutput.cs`
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/OddsScheduleClient.cs`
- `dai/platform/dotnet/DevCore.Domain/Sports/Sport.cs`
- `dai/platform/dotnet/DevCore.Domain/Sports/Team.cs`
- `dai/platform/dotnet/DevCore.Data/SeedData/SportsSeedData.cs`
- `dai/platform/dotnet/DevCore.Data/SeedData/CollegeFootballSeedData.cs` (new)
- `dai/platform/dotnet/DevCore.Data/SeedData/CollegeBasketballSeedData.cs` (new)
- `dai/platform/dotnet/DevCore.Data/Migrations/20260419211500_MakeCompetitionFirstClass.cs` (new)
- `dai/platform/dotnet/DevCore.Data/Migrations/AppDbContextModelSnapshot.cs`
- `dai/services/agent-service/app/models/sports.py`
- `dai/services/agent-service/app/routes/sports.py`
- `dai/services/agent-service/app/services/sports_analyzer.py`
- `dai/scripts/dev/sports/test-sports-dev.ps1`
- `dai/scripts/dev/sports/README.md`

**vault docs changed in this slice:**
- `02 Platform/architecture/current-sports-analysis-flow.md`
- `02 Platform/architecture/orchestration.md`
- `02 Platform/architecture/current-agent-run-contract.md`
- `04 Products/sports-v1/ui-concept.md`
- `04 Products/sports-v1/v1-scope.md`
- `06 Execution/roadmap/sports-v1-roadmap.md`
- this handoff

---

### 13. app shell + nav slice

added a persistent app shell with a sticky header and three routed pages. the analyzer page is unchanged in behavior; the app root is now a thin shell.

**what was implemented:**

- `App` (root component) — stripped to shell: imports `RouterOutlet` + `AppHeaderComponent` only. all analyzer logic extracted into a dedicated feature component.
- `AppHeaderComponent` (`app-header/`) — standalone sticky glass header. 64px mobile / 68px desktop. brand links to `/analyzer`. 3-item nav: Matchup Analyzer, History, Account. active state via `routerLinkActive` — no hardcoded boolean. mobile hamburger toggles a slide-down drawer; tapping a nav link closes it.
- `AnalyzerComponent` (`analyzer/`) — extracted from the former `App` class. identical logic and template. selector `app-analyzer`. lazy-loaded at `/analyzer`.
- `HistoryComponent` (`history/`) — mock history page. six realistic sample reads (NFL, NBA, MLB mix). sport badges, confidence bands, grounded signal chips. filter tabs: All reads / Saved reads. save/unsave toggle is client-side signal state only — no backend persistence.
- `AccountComponent` (`account/`) — mock account page. four sections: Profile, Plan, Delivery, Security. all data is static placeholder. no real auth, billing, or delivery wiring.
- `app.routes.ts` — four routes: `''` redirects to `analyzer`; `/analyzer`, `/history`, `/account` each lazy-load their component.
- `app.spec.ts` — updated: removed analyzer-heading test (heading now in child route); added `provideRouter(routes)`.

**Saved Reads placement:** not a top-level nav item. lives inside History as a filter tab (`All reads` / `Saved reads`). the top nav stays at three items.

**what was NOT built:**
- real run history api or history data binding — History reads are client-side mock only
- real account management, auth, or billing — Account is a static shell
- saved reads persistence — save toggle is in-memory, lost on refresh
- any delivery, scheduling, or subscription flows

**files changed:**
- `dai/apps/sports-app/src/app/app.ts` — stripped to shell
- `dai/apps/sports-app/src/app/app.html` — stripped to shell (`<app-header />` + `<router-outlet />`)
- `dai/apps/sports-app/src/app/app.routes.ts` — redirect + three lazy routes added
- `dai/apps/sports-app/src/app/app.spec.ts` — updated to match shell
- `dai/apps/sports-app/src/app/app-header/app-header.component.ts` (updated) — nav items changed to 3; `RouterLink` + `RouterLinkActive` added; `active`/`disabled` booleans removed
- `dai/apps/sports-app/src/app/app-header/app-header.component.html` (updated) — nav items use `[routerLink]` + `routerLinkActive`; brand is now `<a>`; mobile links close drawer on click
- `dai/apps/sports-app/src/app/app-header/app-header.component.scss` (updated) — `<a>` reset styles added
- `dai/apps/sports-app/src/app/analyzer/analyzer.component.ts` (new)
- `dai/apps/sports-app/src/app/analyzer/analyzer.component.html` (new)
- `dai/apps/sports-app/src/app/analyzer/analyzer.component.scss` (new)
- `dai/apps/sports-app/src/app/history/history.component.ts` (new)
- `dai/apps/sports-app/src/app/history/history.component.html` (new)
- `dai/apps/sports-app/src/app/history/history.component.scss` (new)
- `dai/apps/sports-app/src/app/account/account.component.ts` (new)
- `dai/apps/sports-app/src/app/account/account.component.html` (new)
- `dai/apps/sports-app/src/app/account/account.component.scss` (new)

**vault docs changed:**
- `04 Products/sports-v1/ui-concept.md` — shell architecture section rewritten; account screen description corrected; history deferral clarified
- `04 Products/sports-v1/v1-scope.md` — new frontend section added distinguishing wired vs mocked pages
- `03 Niches/sports-analytics/decisions/0002-three-item-nav-shell.md` (new) — decision note for nav structure

---

### 12. typed EvaluatorOutput + calibrated confidence slice

introduced the evaluate step as an explicit typed contract. confidence ownership transferred from the analyzer to the evaluator. three-step pipeline fully established.

**the problem that was fixed:**
`AgentRunExecutionResult.Confidence` previously passed through `analyzerOutput.Confidence` unchanged. the analyzer's confidence measures internal narrative consistency of the model's reasoning from its inputs — not signal quality, not evidence richness. a model with no grounded data can claim 0.80 confidence because its story is coherent. that number was being stored and returned as the final confidence with nothing distinguishing it from a grounded reading.

**what was implemented:**

- `EvaluatorOutput` (new, `DevCore.Api/AgentRuns/`) — typed record with `AggregateConfidence` (calibrated), `AnalyzerConfidence` (original model estimate, stored for learning loop comparison), and `ConfidenceBand` (`"high"` / `"medium"` / `"low"`).
- `AgentRunService.Evaluate` (new private static method) — scores grounded evidence richness against analyzer confidence. pure computation, no i/o. two-state calibration:
  - zero grounded signals: dampen by 0.75, clamp to [0.30, 0.60]. covers NBA, and NFL/MLB when APIs fail.
  - one or more grounded signals: clamp to [0.35, 0.85]. covers NFL with spread, MLB with starters.
- `AgentRunService.SportMaxGroundedSignals(sport)` (new private static method) — returns the current maximum grounded signals possible per sport under current collection config. makes the zero-vs-one state meaningful relative to what the system can retrieve. update when a new source is added.
- `AgentRunService.ExecuteSportsMatchupAsync` — evaluate step added between analyze and compose.
- `AgentRunService.ComposeDecisionArtifact` — signature updated to `(SportsAnalysisResponse, SportsCollectorOutput, EvaluatorOutput)`. `Confidence` now comes from `evaluator.AggregateConfidence`. `AnalyzerConfidence` added to the result for the learning loop.
- `AgentRunContracts.cs` — `AgentRunExecutionResult` gains `double? AnalyzerConfidence = null`. `Confidence` comment updated to document evaluator ownership and calibration parameters.
- `SportAnalysisContracts.cs` — `SportsAnalysisResponse.Confidence` comment updated: it is the analyzer's provisional input to `Evaluate`, not the final value.

**calibration model (honest proxy):**

| state | condition | formula | result range | band |
|---|---|---|---|---|
| no evidence | groundedCount == 0 or maxGrounded == 0 | analyzerConf × 0.75, clamp [0.30, 0.60] | 0.30–0.60 | low or medium |
| grounded | groundedCount >= 1 | analyzerConf, clamp [0.35, 0.85] | 0.35–0.85 | any |

examples with typical model outputs:
- NBA, analyzer says 0.80 → 0.80 × 0.75 = 0.60 → clamped to 0.60 → medium
- NBA, analyzer says 0.50 → 0.50 × 0.75 = 0.375 → low
- NFL with spread, analyzer says 0.80 → clamped to 0.80 → high
- NFL offseason (no spread), analyzer says 0.80 → 0.60 → medium
- MLB with starters, analyzer says 0.75 → 0.75 → medium/high boundary

**what was NOT built:**
- per-category signal weights (requires learning loop outcome data to validate)
- sport-specific scoring configs (profile system deferred)
- `ConfidenceBand` surfaced to the UI (stored in OutputJson for the learning loop, not in the API response)
- fake high-precision confidence formulas (dampening + clamp is the honest model for now)

**files changed:**
- `DevCore.Api/AgentRuns/EvaluatorOutput.cs` (new)
- `DevCore.Api/AgentRuns/AgentRunService.cs` — evaluate step added; `Evaluate` + `SportMaxGroundedSignals` methods added; `ComposeDecisionArtifact` signature updated
- `DevCore.Api/AgentRuns/AgentRunContracts.cs` — `AnalyzerConfidence` added to `AgentRunExecutionResult`; `Confidence` comment updated
- `DevCore.AiClient/SportAnalysisContracts.cs` — `SportsAnalysisResponse.Confidence` comment updated

**vault docs changed:**
- `orchestration.md` — current truth rewritten for three-step pipeline; confidence ownership section updated; transition path updated

---

### 11. typed SportsCollectorOutput slice

promoted the ad hoc collect step into an explicit typed contract. fixed `GroundedSignals` ownership — it now lives in the collector, not the analyzer. `ComposeDecisionArtifact` seam strengthened to receive both outputs.

**the problem that was fixed:**
`GroundedSignals` was previously set by FastAPI's `analyze_nfl` and `analyze_mlb` after the model call, using `model_copy(update={"groundedSignals": grounded})`. it rode back on `SportsAnalysisResponse` even though the model cannot know what was retrieved. this put evidence provenance in the wrong layer — the analyzer's output type — where it had no business being.

**what was implemented:**

- `SportsCollectorOutput` (new, `DevCore.Api/AgentRuns/`) — typed record with `NflMarketContext?`, `MlbStarterContext?`, and `GroundedSignals[]` computed from the non-null fields. this is the single source of truth for which signals had real retrieved data.
- `AgentRunService.CollectAsync` (new private method) — extracts the sport-scoped retrieval calls into their own named step. returns `SportsCollectorOutput`. makes the two-step flow explicit: collect → analyze.
- `AgentRunService.ComposeDecisionArtifact` updated to receive `(SportsAnalysisResponse, SportsCollectorOutput)`. reads `GroundedSignals` from the collector, not from the analyzer output. the future evaluator input arrives as a third parameter when scoring exists.
- `SportsAnalysisResponse` (.NET + Python) — `groundedSignals` removed. the analyzer output now contains only what the model produced: `lean`, `summary`, `confidence`, `factors`.
- `analyze_nfl` and `analyze_mlb` in `sports_analyzer.py` — `model_copy(update={"groundedSignals": ...})` removed. these functions return the raw model result. evidence provenance is the platform layer's concern.

**ownership clarity delivered:**
- model output: `lean`, `summary`, `confidence`, `factors` — what the model reasoned from the inputs it was given
- collector output: `NflMarketContext?`, `MlbStarterContext?`, `GroundedSignals[]` — what was retrieved before the model was called
- decision artifact: produced by `ComposeDecisionArtifact` from both — today mostly pass-through; future: scored confidence from the evaluator

**what was NOT built:**
- typed evaluator output
- static confidence calibration rule
- source status objects (null/non-null on context fields is the status for now)
- `CollectedAt` timestamp (NflMarketContext.UpdatedAt carries bookmaker timestamp where relevant)

**files changed:**
- `DevCore.Api/AgentRuns/SportsCollectorOutput.cs` (new)
- `DevCore.Api/AgentRuns/AgentRunService.cs` — `CollectAsync` extracted; `ComposeDecisionArtifact` signature updated
- `DevCore.AiClient/SportAnalysisContracts.cs` — `GroundedSignals` removed from `SportsAnalysisResponse`
- `dai/services/agent-service/app/models/sports.py` — `groundedSignals` removed from `SportsAnalysisResponse`
- `dai/services/agent-service/app/services/sports_analyzer.py` — `model_copy` calls removed from `analyze_nfl` and `analyze_mlb`; `groundedSignals=[]` removed from `_parse_response`

**vault docs changed:**
- `orchestration.md` — current truth updated; `ComposeDecisionArtifact` signature updated; transition path step 4 marked done
- `current-sports-analysis-flow.md` — flow map updated; `AgentRunService` section updated; `SportsCollectorOutput` section added; FastAPI section updated

---

### 10. mlb starting pitcher injection slice

added real probable starter data from `statsapi.mlb.com` to the MLB analysis prompt. eliminates the fabricated pitcher name, ERA comparison, and bullpen workload claims that were identified as the second-highest fabrication risk in the evaluation gate.

**what changed:**

- `MlbStarterClient` (new, `DevCore.Api/Sports/`) — typed `HttpClient` that calls `GET /api/v1/schedule?sportId=1&date=YYYY-MM-DD&hydrate=probablePitcher`. caches results 30 minutes (starters are more stable than market lines). team name matching: exact first, then contains-based fallback (handles "Red Sox" matching "Boston Red Sox"). returns `MlbStarterContext?` — null if either starter is not yet announced, team name mismatch, or any api failure.
- `MlbStarterContext` (new, `DevCore.AiClient/`) — record carrying `HomeStarterName`, `HomeStarterHand`, `AwayStarterName`, `AwayStarterHand`. handedness codes match MLB Stats API `pitchHand.code`: `"L"`, `"R"`, `"S"`. defined in the ai client layer; serialized as `mlbStarterContext` (camelCase) by web defaults.
- `SportsAnalysisRequest` extended — new optional `MlbStarterContext? MlbStarterContext = null` field.
- `AgentRunService` — for MLB runs: calls `MlbStarterClient.GetStartersAsync()` before building the ai request; attaches result (or null) to `SportsAnalysisRequest`.
- `Program.cs` — `MlbStarterClient` registered with `AddHttpClient<MlbStarterClient>` pointing to `statsapi.mlb.com`.
- `app/models/sports.py` — `MlbStarterContext` Pydantic model added; `SportsAnalysisRequest` extended with `mlbStarterContext: MlbStarterContext | None = None`.
- `app/services/sports_analyzer.py`:
  - `analyze_mlb` now accepts `starter_context: MlbStarterContext | None`
  - when present: injects a `[starter data]` block — home and away pitcher name and handedness — and instructs the model to note handedness advantages. instructs the model NOT to fabricate ERA or recent form.
  - when absent: injects `[no starter data available]` and explicitly instructs the model to omit the starting pitching category entirely.
  - `_MLB_SYSTEM` updated: removed `"name probable starters and note any edge in era, recent form, or handedness vs lineup"` (the direct fabrication cause); replaced with a conditional instruction gated on `[starter data]`.
  - `groundedSignals=["starting_pitching"]` when real starter data was injected; `[]` otherwise.
- `app/routes/sports.py` — passes `req.mlbStarterContext` through to `analyze_mlb`.

**fabrication eliminated:**
- before: `_MLB_SYSTEM` instructed the model to "name probable starters and note any edge in era, recent form, or handedness vs lineup" — unconditionally. model invented names and stats from training weights.
- after: system prompt is conditional on the `[starter data]` block. when no data: model cannot reference starting pitching at all. when data present: model uses only the confirmed name and handedness injected into the prompt.

**scope explicitly deferred:**
- ERA, recent form stats — would require rotowire or additional MLB Stats API fields; deferred
- bullpen workload — no real data source in this slice; model may still use prior knowledge for bullpen/lineup/ballpark categories (structural facts with lower fabrication risk than player-specific stats)
- weather — MLB outdoor ballpark weather not in scope for this slice
- mlb market enrichment (line/spread) — not in scope

**ownership note:** starter data retrieval lives in .NET (not FastAPI) for the same reason as `OddsMarketClient`: deterministic retrieval belongs in the platform layer. the MLB Stats API is free and public — no api key required.

**files changed:**
- `DevCore.AiClient/MlbStarterContext.cs` (new)
- `DevCore.AiClient/SportAnalysisContracts.cs` — `MlbStarterContext?` added to `SportsAnalysisRequest`
- `DevCore.Api/Sports/MlbStarterClient.cs` (new)
- `DevCore.Api/Program.cs` — `MlbStarterClient` registered
- `DevCore.Api/AgentRuns/AgentRunService.cs` — `MlbStarterClient` injected; MLB starter lookup added
- `dai/services/agent-service/app/models/sports.py` — `MlbStarterContext` model + `mlbStarterContext` field added
- `dai/services/agent-service/app/services/sports_analyzer.py` — `analyze_mlb` updated; `_MLB_SYSTEM` updated
- `dai/services/agent-service/app/routes/sports.py` — starter context passed to `analyze_mlb`

---

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
  → GET /api/competitions
      ← CompetitionReferenceDto[] { code, displayName, sportFamily, level, isSupported, availabilityNote, ... }
  [user selects sport family + level]
  → GET /api/competitions/{code}/teams
  → GET /api/competitions/{code}/matchup-dates?teamA=X&teamB=Y
      → OddsScheduleClient.GetEventsAsync() (non-directional, cached 30min)
      ← MatchupEventDto[] { date, homeTeam, awayTeam }
  [user selects date pill → selectedEvent resolved]
  → POST /api/agent-runs
    → AgentRunsController.Create()
      → IdentityResolver (dev bypass: TenantKey=1, UserKey=1)
      → INSERT AgentRun (status=pending, CorrelationId=TraceIdentifier)
        → AgentRunService.ExecuteAsync(req, run.AgentRunId, ct)
        → CollectAsync(input, gameDate, ct)
            [nfl only] → OddsMarketClient.GetNflSpreadAsync()
            [mlb only] → MlbStarterClient.GetStartersAsync()
            [nba / ncaamb] → EspnBasketballScheduleClient.GetRestContextAsync()
            [ncaaf] → no grounded collector yet
          ← SportsCollectorOutput { NflMarketContext?, MlbStarterContext?, BasketballScheduleContext?, GroundedSignals[] }
        → FastApiClient.AnalyzeSportsMatchupAsync(req, agentRunId, ct)
          → POST http://127.0.0.1:8001/api/sports/analyze  ← port 8001 (fresh); port 8000 is a stale process that cannot be killed from a different terminal session
              X-Correlation-Id: Activity.Current?.Id
              X-Agent-Run-Id: AgentRun.AgentRunId   ← cross-layer anchor
            → sports.py: validates competition, logs agent_run_id + correlation_id
            → sports_analyzer.py: gpt-4o-mini, family-aware prompt routing, temperature=0.3
          ← SportsAnalysisResponse { lean, summary, confidence, factors }
        → Evaluate(analyzerOutput, collector, competition)
          ← EvaluatorOutput { aggregateConfidence, analyzerConfidence, confidenceBand }
      → UPDATE AgentRun (completed, OutputJson, DurationMs)
      ← AgentRunResultDto { agentRunId, status, lean, summary, confidence, factors, createdUtc, durationMs }
  ← render: hero / control rail / Matchup Read / Factor Breakdown
```

### what is truly working today

- competition-aware SQL reference data (Sports + Teams tables)
- visible competition matrix with explicit support state:
  - football + pro → `nfl`
  - football + college → `ncaaf`
  - basketball + pro → `nba`
  - basketball + college → `ncaamb`
  - baseball + pro → `mlb`
  - baseball + college → unavailable
- selector flow is now `sport` → `level` → teams → date → analyze
- OddsAPI schedule lookup with non-directional cache
- college football and college basketball schedule lookup via odds api competition keys
- **NFL market enrichment:** `OddsMarketClient` fetches current spread before each NFL analysis; result injected into the user message as a `[market data]` block; null result triggers no-market fallback that explicitly prevents fabrication
- **basketball rest / schedule enrichment:** `EspnBasketballScheduleClient` fetches recent prior-game dates from public espn scoreboard data for `nba` and `ncaamb`; when both teams are grounded, the analyzer receives a `[schedule data]` block with days rest and back-to-back flags; when not, it receives an explicit no-data fallback and must omit rest claims
- **MLB starter enrichment:** `MlbStarterClient` fetches probable starter names and handedness from `statsapi.mlb.com` before each MLB analysis; result injected as a `[starter data]` block; null result triggers no-starter fallback that omits the starting pitching category and prevents fabrication of pitcher names, ERA, or form
- **typed collect step:** both retrieval calls flow through `SportsCollectorOutput`. `GroundedSignals` computed from context availability, not by the model.
- **typed evaluate step:** `EvaluatorOutput` produced by `AgentRunService.Evaluate` after the analyzer call. calibrated confidence replaces the analyzer's provisional value in the decision artifact. `AnalyzerConfidence` stored separately for learning loop comparison.
- AgentRun persistence: pending → completed/failed with InputJson, OutputJson, DurationMs, CorrelationId
- FastAPI sports analysis: competition-aware routing with football / basketball / mlb analyzer families, gpt-4o-mini, structured JSON response with `lean`, `summary`, `confidence`, `factors[]`
- `lean` field: one-sentence directional signal, propagated through all contract layers, rendered in the `Current Lean` module
- `X-Agent-Run-Id` forwarded on every sports call; FastAPI logs it
- `durationMs` and `agentRunId` surfaced in the Analysis Details block of the UI
- `AgentRunService` dispatches by `RunType`; `"sports.matchup.analysis"` routes correctly; unknown types fail loudly with a stored error

### what is scaffolded / not yet active

- `AgentProfileKey` — stored null, no profile loaded at runtime
- `CreateAgentRunRequest.Input` typed as `CompetitionMatchupInput` — still matchup-analysis-specific and will need generalization for a second run type
- `GET /api/agent-runs/{id}` — endpoint exists, works, UI never calls it
- competition-specific college collectors, evaluator rules, and prompt specializations beyond family-level routing — not built yet

---

## what docs were corrected

| doc | what changed |
|---|---|
| `current-agent-run-contract.md` | request contract updated from `sport` to `competition`; internal FastAPI contract updated; OutputJson example updated with calibrated-confidence fields |
| `current-sports-analysis-flow.md` | flow map rewritten around `/api/competitions`; sport family + level selector flow documented; supported competition matrix and unsupported college baseball state added |
| `orchestration.md` | current truth updated from sport to competition routing; `CompetitionMaxGroundedSignals` documented; next-step guidance updated after the competition-first slice |
| `ui-concept.md` | selector flow updated to sport + level; picker-system decision extended to level; outward-facing label rules updated for sport family, level, and competition |
| `v1-scope.md` | current dev slice note updated for competition-aware support; NCAAF and NCAAMB moved into scope; college baseball remains out of scope |
| `sports-v1-roadmap.md` | top-line product framing updated for current competition-aware support; immediate next steps updated; post-v1 list no longer treats MLB or college football/basketball as future expansion |
| this handoff | competition-first slice added as section 14; current truth and next-step guidance updated |
| `ui-concept.md` | visual system section replaced with dark token system and four-tier card hierarchy; interaction language updated with `.inset-surface`; hierarchy decisions block updated with eyebrow removal and divider system; visual direction note updated to locked; deferred table dark mode toggle corrected |
| this handoff | dark mode redesign session added as section 4; decisions already made updated with dark direction decisions; remaining open (ui) updated with touch hover validation |

---

## architecture gaps (current priority order)

1. `CreateAgentRunRequest.Input` typed as `CompetitionMatchupInput` — still matchup-analysis-specific; will need to become a generic envelope before a second run type can be added. documented with a comment in the record.
2. `OutputJson` has no schema — stored as raw JSON blob; cannot be queried or validated structurally
3. no auth on `AgentRunsController` — dev bypass only; not production-safe
4. `useStubApi = true` in `environment.ts` production build — real API only called in dev
5. `GET /api/agent-runs/{id}` — endpoint exists and works; UI still never calls it; wiring it up is the next natural step for making individual runs inspectable
6. no run history in UI — runs accumulate in DB with no visibility surface. the History page shell now exists as a client-side mock; wiring it to real api data (GET /api/agent-runs) is the natural follow-on.
7. `ncaaf` still has no grounded collector — it routes cleanly, but it remains permanently zero-grounded until a real evidence source is added

---

## next slice candidates

orchestrator-ready foundation is in place. seam named. evidence quality tracked. agent/tool doctrine written. competition is now first-class in the selector flow and request path.

**recommended next: remove the last permanently zero-grounded supported competition.**

the three-step pipeline (collect → analyze → evaluate) is now fully established. all three typed contracts exist. calibrated confidence is in place with an honest proxy model. the cleanest follow-on is to make the new competition seam matter for evidence quality, not just routing.

**option A: college-football market grounding for `ncaaf`.** this extends the existing NFL market pattern to the only supported competition that still has no grounded source at all. it is the cleanest way to stop every `ncaaf` run from being evaluator-dampened by design.

**option B: basketball injury grounding for `nba` and `ncaamb`.** rest/schedule is now grounded, but basketball still has no deterministic injury input. this would add a second signal category without reopening the UI or changing the three-step seam.

**option C: outcome tracking foundation.** the calibration parameters (0.75 dampening, clamp ranges) are still conservative estimates. the learning loop needs game outcomes to compare against stored `Confidence` and `AnalyzerConfidence`. this is strategically important, but it is broader and less aligned to the just-finished collector slice.

**recommendation: option A (`ncaaf` market grounding).** it removes the final always-zero-grounded supported competition, reuses an already-proven collector pattern, and keeps the competition seam doing real platform work instead of only selection/routing work.

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
- one-off selector controls — sport and level use the shared picker family; do not split them back out into separate one-off controls
- fabricated lean or pick text — neutral fallback only
- visual direction — dark navy workspace, cobalt accent, four-tier card depth hierarchy. locked. do not reintroduce light or warm surfaces.
- page background — `body::before` fixed layer, clean linear gradient only. no radial glow layers.
- eyebrow pill labels — removed. `Matchup Read` and `Factor Breakdown` stand alone as section titles. do not re-add pill labels as section chrome.
- divider system — `border-b border-[#22344a] pb-4 lg:pb-5` under all four main section headers. one class, consistent across the page.
- hover tiers — `.premium-surface` for factor cards and lean card; `.inset-surface` for stat cells. do not apply the same class to both surface types.

**nav / shell:**
- top nav is 3 items: Matchup Analyzer, History, Account. do not expand the top nav without a clear product reason.
- Saved Reads is not a top-level nav item. it lives inside History as a filter tab. do not move it back to the top nav.
- active state is driven by `routerLinkActive`. do not reintroduce hardcoded active booleans.
- History and Account are mock pages in the same design language as the analyzer. do not reduce them to dead placeholders or empty routes.

**architecture:**
- `X-Agent-Run-Id` as the cross-layer correlation anchor — settled
- `durationMs` in the API response — in place

---

## active implementation files

**Angular:**
- `dai/apps/sports-app/src/app/app.ts` — root shell (RouterOutlet + AppHeaderComponent only)
- `dai/apps/sports-app/src/app/app.html` — shell template
- `dai/apps/sports-app/src/app/app.routes.ts` — route definitions
- `dai/apps/sports-app/src/app/app-header/app-header.component.ts` — sticky header, 3-item nav
- `dai/apps/sports-app/src/app/app-header/app-header.component.html`
- `dai/apps/sports-app/src/app/app-header/app-header.component.scss`
- `dai/apps/sports-app/src/app/analyzer/analyzer.component.ts` — matchup analyzer page (extracted from former app.ts)
- `dai/apps/sports-app/src/app/analyzer/analyzer.component.html`
- `dai/apps/sports-app/src/app/history/history.component.ts` — mock history page
- `dai/apps/sports-app/src/app/history/history.component.html`
- `dai/apps/sports-app/src/app/history/history.component.scss`
- `dai/apps/sports-app/src/app/account/account.component.ts` — mock account page
- `dai/apps/sports-app/src/app/account/account.component.html`
- `dai/apps/sports-app/src/app/account/account.component.scss`
- `dai/apps/sports-app/src/app/core/models/agent-run.model.ts`
- `dai/apps/sports-app/src/app/sports-api.service.ts`
- `dai/apps/sports-app/src/app/team-picker/team-picker.component.ts`
- `dai/apps/sports-app/src/app/team-picker/team-picker.component.html`
- `dai/apps/sports-app/src/styles.css`

**.NET:**
- `dai/platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs`
- `dai/platform/dotnet/DevCore.Api/Controllers/SportsReferenceController.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunService.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/SportsCollectorOutput.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/EvaluatorOutput.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/IAgentRunService.cs`
- `dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs`
- `dai/platform/dotnet/DevCore.AiClient/FastApiClient.cs`
- `dai/platform/dotnet/DevCore.AiClient/BasketballScheduleContext.cs`
- `dai/platform/dotnet/DevCore.AiClient/SportAnalysisContracts.cs`
- `dai/platform/dotnet/DevCore.AiClient/NflMarketContext.cs`
- `dai/platform/dotnet/DevCore.AiClient/MlbStarterContext.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/CompetitionCatalog.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/OddsScheduleClient.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/OddsMarketClient.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/MlbStarterClient.cs`
- `dai/platform/dotnet/DevCore.Api/Sports/EspnBasketballScheduleClient.cs`
- `dai/platform/dotnet/DevCore.Data/SeedData/SportsSeedData.cs`
- `dai/platform/dotnet/DevCore.Data/SeedData/CollegeFootballSeedData.cs`
- `dai/platform/dotnet/DevCore.Data/SeedData/CollegeBasketballSeedData.cs`
- `dai/platform/dotnet/DevCore.Data/Migrations/20260419211500_MakeCompetitionFirstClass.cs`
- `dai/platform/dotnet/DevCore.Data/Migrations/AppDbContextModelSnapshot.cs`

**FastAPI:**
- `dai/services/agent-service/app/routes/sports.py`
- `dai/services/agent-service/app/models/sports.py`
- `dai/services/agent-service/app/services/sports_analyzer.py`

---

## remaining open (ui)

- browser-verify empty, populated, mobile, and long-scroll states in a real browser (in dark direction)
- validate picker behavior across mouse, keyboard, and touch
- validate hover states on touch devices — `.premium-surface` and `.inset-surface` are wrapped in `@media (hover: hover)` and should not apply on touch; confirm no unintended static state change on mobile
- verify hover states on touch devices once tested in a real mobile browser — `.premium-surface` and `.inset-surface` are wrapped in `@media (hover: hover)` and should not apply on touch

## remaining open (platform)

- apply the competition seed migration to the running DB instance
- apply `RenameOaklandToSacramento` migration to running DB instance
- auth on `AgentRunsController` before any production deployment
- `useStubApi` in `environment.ts` — confirm production build value before any real deployment
