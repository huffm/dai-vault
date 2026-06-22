# Schedule/Fatigue Context v1 — MLB Bullpen Adapter

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code change (dai). Backward compatible. No confidence/
posture/lean logic, prompt-doctrine, buyer-UI, reconciliation, or migration change. No new paid/fragile
providers (reuses free public statsapi).
**implements:** the recommended next slice from `availability-context-v1.md`; second new source lane on the
Cross-Sport Source Envelope contract.

## What shipped

The first **FatigueContext** lane: an MLB **bullpen-workload proxy** emitting a `bullpen_availability`
`SourceSignalEnvelope`. Bullpen/fatigue is now an observed-or-honestly-unavailable proxy signal rather
than an ungrounded named risk.

- **`MlbFatigueContext`** (`DevCore.AiClient`, new) -- per-team relievers-used-last-game + a heavier-
  workload side (home/away/neutral/unknown) + retrieved-at timestamp.
- **`MlbBullpenFatigueClient`** (`DevCore.Api/Sports`, new) -- resolves each team's id (cached statsapi
  teams list), finds its most recent FINAL game in the prior 7 days, and reads the boxscore pitcher count
  (relievers = pitchers used minus the starter). Fail-soft at every step; null when neither team resolves.
  30-minute cache.
- **Gateway tool** `fatigue.mlb.bullpen` -- new `ToolIds` const + `ToolRegistry` entry (Retriever,
  HttpExternal, allowed node `platform.retrieve`, Free, 30-min ttl) + `FatigueMlbBullpenHandler` + DI
  registration + `AddHttpClient<MlbBullpenFatigueClient>` (statsapi base). Mirrors the availability tool.
- **`SportsRetriever`** -- in the MLB block, invokes the bullpen tool by matchup; threads the result into
  SourceDepth + the envelope builder.
- **`SourceDepthEvaluator`** -- additive `bullpen_availability` depth: resolved proxy -> `shallow`
  (derived, never enriched); unresolved -> no record (never inflates depth/breadth).
- **`SourceEnvelopeBuilder`** -- additive bullpen envelope.

## MLB bullpen adapter behavior

A lightweight, team-level, recent-games PROXY. It counts relievers used in each team's most recent
completed game (boxscore pitchers minus the starter) and names the heavier-workload side. It does NOT
make bullpen-quality claims, does NOT project individual reliever availability, and computes no advanced
metrics -- nothing invented.

- both teams resolved -> **proxy** (depth shallow), side = the team that used more relievers last game.
- one team resolved -> **proxy** (partial), side = unknown.
- neither resolved (no recent final game, http failure, no gamePk/pitcher list) -> **unavailable**.

## Existing basketball rest mapping

**Deferred** (documented follow-up, per the slice's allowance). The NBA/NCAAB `rest_schedule` signal
already exists via `EspnBasketballScheduleClient` but is not yet mapped into a `rest_travel` envelope;
mapping it touches the basketball retrieval path and is a clean standalone slice. The cross-sport
`rest_travel` envelope shape is reserved and ready (see Cross-Sport reuse).

## Envelope fields (bullpen_availability)

`SignalKey=bullpen_fatigue`, `SourceGroup=bullpen_availability`, `Sport=mlb`, `Provider=mlb_statsapi`
(null when unavailable), `Status` (proxy when resolved, unavailable otherwise), `Depth` (shallow when
resolved, else none), `Observed`, `UpdatedAtUtc` + `FreshnessSummary`, `ArtifactSafeSummary`
("{side} bullpen more taxed recently"), `ModelContextSummary` (relievers-used home/away + side).

## Status / depth behavior

Statuses reuse the existing `SourceSignalStatuses` vocabulary (no new status). A recent-games proxy is
always **proxy** when resolved (never `observed` -- it is derived, not a direct fatigue read), and
`bullpen_availability` depth is always **shallow** (never enriched). Unresolved -> unavailable + no depth
record. Bullpen is NOT added to `GroundedSignals`/`SignalAvailability`, so `SourceSufficiency` breadth and
confidence are untouched (observed-only depth this slice).

## Freshness / provenance

`RetrievedAtUtc` (boxscore/schedule fetch time) is carried as `UpdatedAtUtc` + `FreshnessSummary`. The
proxy is derived from each team's most recent final game.

## Fallback behavior

Fail-soft throughout: an unparseable teams list, an unresolved team id, no recent final game, or any http
failure yields null or a partial context -> the envelope reports `unavailable` and the run proceeds
unchanged. The run never fails because the bullpen proxy is absent.

## Tests

- `SourceEnvelopeBuilderTests` +3: null->unavailable, both->proxy/shallow (+side), partial->proxy.
- `SourceDepthEvaluatorTests` +2: resolved->shallow bullpen_availability, null/unresolved->no record.
- `SportsRetrieverTests` updated: registered the bullpen tool/handler in the test gateway (empty teams ->
  unavailable, breadth unchanged).
- Full `DevCore.Api.Tests` suite: **793 passed / 0 failed** (was 788; +5). No regressions.
- No Python/frontend changes -> no py_compile / sports-app runs needed.

## Verification artifact (live)

Real MLB run `deb5433e` (2026-06-22): completed; `sourceEnvelopes` carried four groups --
`starting_pitching` observed/enriched, `market_odds` observed/enriched, `lineup_injury` proxy/shallow, and
**`bullpen_availability` proxy/shallow (mlb_statsapi, "away bullpen more taxed recently")**. The adapter
resolved both teams' recent relief usage and named the heavier side. `sourceDepth` showed
`bullpen_availability=shallow` (not overstated). lean away @ 0.8, posture monitor -- unchanged.

## Cross-sport reuse notes (NBA/NFL)

The fatigue/rest envelope shapes are sport-agnostic; future adapters drop in without touching the
envelope, SourceDepth seam, or the artifact surface:
- **NBAScheduleFatigueAdapter** -> `rest_travel` envelope (back-to-back, days rest, travel) -- the
  existing ESPN rest signal maps straight in (deferred mapping above).
- **NFLShortWeekFatigueAdapter** -> `rest_travel` / `bullpen_availability`-analog (short week, rest
  differential, snap-load proxy later).
Each emits its envelope from its provider; the cross-sport contract is unchanged.

## Deferred items

- Basketball `rest_schedule` -> `rest_travel` envelope mapping (low risk; standalone follow-up).
- Player-level reliever availability (no reliable free source) -- team-level relievers-used only.
- Relief-innings and multi-game (last-3) workload (single most-recent game this slice).
- Analyzer prompt-injection of the fatigue summary (kept prompt doctrine unchanged; data-block injection
  is a documented follow-up).
- `stale` status (freshness captured; threshold engine deferred).
- NBA/NFL fatigue adapters; named-risk-grounding integration (bullpen/fatigue risks are now groundable).

## Recommended next slice

**Named-Risk Grounding Integration v1** -- now that lineup_injury and bullpen_availability are groundable
source lanes, wire the named-risk-grounding projection to map availability/fatigue named-risks onto them
(the original motivation behind these lanes). Alternatively **VenueEnvironment Context v1**
(`weather_park`), **Market Agreement Projection v1**, or **Reconcile Directional-Contrast Cohort v1** once
games 200013-200022 are Final.
