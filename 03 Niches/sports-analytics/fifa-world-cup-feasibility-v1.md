# FIFA World Cup Soccer -- Feasibility Spike v1

**date:** 2026-06-28
**status:** READ-ONLY feasibility spike. No code changed, no soccer support added, no buyer surface, no Cognitive
Factory activation. Conclusion: **CONDITIONAL-GO for an internal sample-capture spike behind a new competition
config; NO-GO for any buyer-facing World Cup support until the three-way market + knockout-semantics gaps are closed.**
**type:** Lane C feasibility spike. dai-skill-router gate + dai-grill-with-vault (architecture fit). Findings are
source-cited and were verified against `dai` source + a live the-odds-api probe; the Graphify/agent map was treated
as navigation evidence only.
**anchor:** Determine whether the existing platform (factory) can represent FIFA World Cup matches as a
competition/niche configuration problem, without hard-coding World Cup business logic into core services.

See also: [[thesis]], [[data-sources]], [[signals]], decisions `0001-football-and-basketball-first`,
`directional-contrast-cohort-v4-market-baseline-v3` (the MLB capture run alongside this spike).

## 1. Current support status

**Not supported; would return 400.** The routable competitions are exactly NFL, NCAAF, NBA, NCAAMB, MLB.

- .NET catalog: `CompetitionCatalog._all` (`platform/dotnet/DevCore.Api/Sports/CompetitionCatalog.cs:53-129`) defines
  5 routable competitions + 1 explicitly-not-supported (college baseball, `OddsApiKey: null`). No soccer entry.
  `CompetitionCodes` (`:3-10`) has no soccer constant.
- Python analyzer gate: `_SUPPORTED_COMPETITIONS = {"nfl","ncaaf","nba","ncaamb","mlb"}`
  (`services/agent-service/app/routes/sports.py:13`); an unsupported competition raises HTTP 400 (`:25-29`).
- Readiness is owned separately from routability: `ReadinessLevels` + `CompetitionDefinition.IsBuyerReady`
  (`CompetitionCatalog.cs:15-44`). Even US sports are mostly `smoke_level_only`; only NBA + MLB are
  `buyer_ready_validated`. This is the right lever to keep World Cup internal-only if added.

## 2. Data-source availability

- **Market odds: AVAILABLE.** Live the-odds-api v4 `/sports` probe (2026-06-28) returns `soccer_fifa_world_cup`
  with **`active: true`**, plus `soccer_fifa_world_cup_winner` (futures). So World Cup match odds are reachable
  through the existing parameterized odds path. (Consistent with the 2026 Round of 32 being live.)
- **Results / reconciliation source: GAP.** MLB finals come from StatsAPI (`mlb_statsapi`), which is baseball-only
  and cannot resolve soccer results. There is **no wired soccer finals source**. the-odds-api has a `/scores`
  endpoint that could serve this, but it is not integrated, and it does not by itself disambiguate regulation vs
  extra-time/penalties vs advancement (see sec 5). A reliable soccer results feed is the single biggest data gap.
- Match-level enrichment (form, injuries, lineups, rest, congestion) has **no provider wired** -- soccer would start
  market-only (richness comparable to football's `["market","sharp_public"]`), not starter-grounded like MLB.

## 3. Identity model fit -- GOOD (soft-coupled)

The canonical reconciliation key is `(SourceProvider, ExternalGameId)` on the `AgentRun` row, and the matcher is
provider-agnostic (`OutcomeReconciliationService.MatchAsync` selects on provider + external id + tenant;
`OutcomeReconciliationMatcher` has no sport branch). MLB happens to use the StatsAPI gamePk as its external id, but
non-MLB sports already key off the odds-event id. A FIFA match maps cleanly to
`(SourceProvider="odds_api", ExternalGameId=<odds event id>)`. No identity schema change required. **Verdict: fits.**

## 4. Market model fit -- HARD GAP (two-way assumption)

The platform's directional + market model assumes a **two-outcome** market:

- `LeanSide` is `home` / `away` / `null` only (`AgentRunContracts.cs:216-220`). There is **no `draw` lean side**.
- Market de-vig is two-way: `ImpliedProbabilityFromAmerican` + a home-vs-away `ConsensusSide` with no draw outcome
  (`MarketDepth.cs`); the cohort de-vig harness normalizes `home+away -> 1.0`. A soccer 1X2 (home/draw/away) market
  would be mis-normalized (the draw probability silently dropped, inflating both sides).
- Odds requests pull `markets=h2h,spreads,totals` (`OddsMarketClient`); soccer h2h on the-odds-api is **three-way**.

**Implication:** the market favorite for a World Cup match cannot be computed correctly by the current two-way de-vig.
Two safe options: (a) use **draw-no-bet** or **to-advance** markets, which are two-way and reuse the existing model
unchanged; or (b) add genuine **1X2 three-way** de-vig + a `draw` lean side (larger change to `LeanSide`,
`MarketDepth`, `ConsensusSide`, and the cohort harness). **Verdict: two-way de-vig is hard-coded; 1X2 needs new code.**

## 5. Reconciliation model fit -- PARTIAL (draw stored, semantics not modeled)

- **Draw is already a first-class outcome.** `OutcomeStatuses.Draw` exists, is in `FinalSettlement`
  (`AgentRunContracts.cs:22,41-49`), and `RunEvaluator.WinningSide("draw")=null -> Evaluate=inconclusive`
  (`RunEvaluator.cs:19-33`). So a drawn match can be recorded and is honestly graded inconclusive against a
  home/away lean. No schema change for draw.
- **The knockout/advancement distinction is NOT modeled.** A run has exactly one outcome; the system cannot
  distinguish *regulation result* vs *winner after extra-time/penalties* vs *to-advance* vs *handicap cover*. For
  a Round-of-32 knockout match these can diverge (regulation draw, advance on penalties). Recording a single
  `home_win/away_win/draw` would silently conflate "what was predicted" with "what resolved." **This must be made
  explicit before any knockout reconciliation** -- the predicted quantity (regulation 1X2 vs to-advance) has to be
  fixed per run and reconciled against the matching result type. **Verdict: storage ready for draw; semantics not
  ready for knockout advancement.**

## 6. Analyzer flow fit -- HARD-CODED to US sports

`POST /api/sports/analyze` dispatches by competition to `analyze_football` / `analyze_basketball` / `analyze_mlb`
(`sports.py:31-60`), each with sport-specific context models and system prompts (`app/services/sports_analyzer.py`,
`app/models/sports.py`). The MLB path is baseball-specific (starting_pitching, bullpen, handedness); there is no
soccer analyzer, soccer system prompt, or soccer context model. A soccer run would need a new `analyze_soccer` path
+ a neutral soccer prompt + a soccer market context. **This is the bulk of the implementation work.** Nothing here
should be retrofitted into the MLB/NBA/NFL paths -- it is a new assembly line, per niche doctrine.

## 7. Required minimal changes (if a spike is approved)

Smallest internal-only path (market-only, two-way market, no buyer surface):

1. `CompetitionCatalog.cs`: add `CompetitionCodes.FifaWorldCup` + a `CompetitionDefinition`
   (`Code="fifa_world_cup"`, `OddsApiKey="soccer_fifa_world_cup"`, `ExpectedSignalNames=["market"]`,
   `IsBuyerReady=false`, `ReadinessLevel=smoke_level_only`).
2. `sports.py`: add `"fifa_world_cup"` to `_SUPPORTED_COMPETITIONS` and a soccer dispatch branch.
3. `sports_analyzer.py` / `models/sports.py`: add `analyze_soccer` + a neutral `_SOCCER_SYSTEM` prompt + a
   `SoccerMarketContext` (no pitcher/inning concepts).
4. **Market**: start with **draw-no-bet or to-advance (two-way)** so the existing de-vig + `LeanSide` model is
   reused unchanged. Defer 1X2 three-way + `draw` lean to a later slice.
5. `OddsMarketClient`: confirm/extend market-type handling for the soccer key (h2h three-way vs DnB/to-advance);
   verify team-name normalization for national teams.
6. **Reconciliation**: fix the predicted quantity per run (regulation result vs to-advance) and wire a soccer results
   source (the-odds-api `/scores` or equivalent) before any settlement.

Safe-to-leave-untouched (sport-generic, verified): `(SourceProvider, ExternalGameId)` identity + matcher; the
outcome+evaluation tables and `RunEvaluator` (already handle draw -> inconclusive); idempotency + the Settlement
Direction Integrity guard; confidence/advertised-strength derivation; artifact composition + pipeline steps.

## 8. Recommended sample-capture plan

**Do NOT capture World Cup samples yet -- it is not safe on the current build.** A soccer run would 400 at the
analyzer gate; forcing it through would mean an MLB-shaped analyzer reasoning about soccer with a two-way market it
cannot de-vig. The clean path is a small, bounded implementation spike (sec 7 steps 1-5) on an isolated branch, then:

- Capture a handful of **pregame Round-of-32 knockout matches**, market-only, two-way (to-advance or DnB), with
  `IsBuyerReady=false`. Record the predicted quantity explicitly. Read artifacts back for identity + integrity.
- Write **no outcomes** until a soccer results source is wired and the predicted-vs-resolved quantity is matched.
- Keep it in its own regime; never pool with MLB calibration.

## 9. Explicit no-go risks

- **No-go: any buyer-facing World Cup support.** Three-way market favorite is currently mis-computable; advertising a
  soccer read would be unsound. Keep `IsBuyerReady=false`.
- **No-go: hard-coding World Cup logic into core services.** Soccer must stay a competition/niche config + a new
  analyzer assembly line; do not branch MLB/NBA/NFL paths on soccer.
- **No-go: settling knockout matches before the predicted quantity (regulation vs to-advance vs after-penalties) is
  explicit and a soccer results source is wired.** Conflating them corrupts the learning signal.
- **No-go: 1X2 three-way de-vig as a first step.** It touches `LeanSide`, `MarketDepth`, `ConsensusSide`, and the
  cohort harness; start two-way (DnB/to-advance) instead.
- **Watch:** national-team name normalization; draws graded inconclusive (a draw-heavy sample yields little
  directional signal); odds availability per match (the World Cup key is active, but per-match h2h depth varies).

## 10. Next-slice prompt (only if implementation is warranted)

> **FIFA World Cup Internal Capture Spike v1 (no buyer surface).** On an isolated branch, add `fifa_world_cup` as a
> non-buyer-ready competition: register it in `CompetitionCatalog` (`OddsApiKey="soccer_fifa_world_cup"`,
> `ExpectedSignalNames=["market"]`, `IsBuyerReady=false`, `ReadinessLevel=smoke_level_only`); add it to the Python
> `_SUPPORTED_COMPETITIONS` with an `analyze_soccer` dispatch + a neutral `_SOCCER_SYSTEM` prompt + a
> `SoccerMarketContext`; use **to-advance or draw-no-bet (two-way)** odds so the existing de-vig + `LeanSide` model
> is reused unchanged. TDD per `dai-test-discipline`; review per `dai-code-reviewer`. Capture a few pregame
> Round-of-32 matches market-only, record the predicted quantity explicitly, read artifacts back for identity +
> integrity, **write no outcomes**. Defer 1X2 three-way de-vig + a `draw` lean side and the soccer results-source
> wiring to separate slices. No buyer copy, no UI, no Cognitive Factory activation, no MLB/NBA/NFL path changes.
