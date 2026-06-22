# Availability Context v1 — MLB First Adapter

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code change (dai). Backward compatible. No confidence/
posture/lean logic, prompt-doctrine, buyer-UI, reconciliation, or migration change. No new paid/fragile
providers (reuses free public statsapi).
**implements:** the recommended next slice from `cross-sport-source-envelope-v1.md`; first new source lane
on the Cross-Sport Source Envelope contract.

## What shipped

The first reusable **AvailabilityContext** lane, MLB adapter first, emitting a `lineup_injury`
`SourceSignalEnvelope`. Lineup availability is now a grounded-or-honestly-missing source signal rather
than an ungrounded named risk.

- **`MlbAvailabilityContext`** (`DevCore.AiClient`, new) -- home/away lineup confirmed flags, lineup
  counts, retrieved-at timestamp.
- **`MlbAvailabilityClient`** (`DevCore.Api/Sports`, new) -- fetches the statsapi boxscore
  (`/api/v1/game/{gamePk}/boxscore`) and reads `battingOrder` presence. Fail-soft: any failure (no gamePk,
  network/http error, unparseable body) returns null. 5-minute cache (lineups change near game time).
- **Gateway tool** `availability.mlb.lineup` -- new `ToolIds` const + `ToolRegistry` entry (Retriever,
  HttpExternal, allowed node `platform.retrieve`, Free, 5-min ttl) + handler `AvailabilityMlbLineupHandler`
  + DI registration + `AddHttpClient<MlbAvailabilityClient>` (statsapi base). Mirrors the starter tool.
- **`SportsRetriever`** -- in the MLB block, after starters resolve the gamePk, invokes the availability
  tool keyed by that gamePk; threads the result into SourceDepth + the envelope builder.
- **`SourceDepthEvaluator`** -- additive `lineup_injury` depth: confirmed lineup(s) -> `shallow` (observed
  presence, never enriched); nothing confirmed -> no record (missing never inflates depth/breadth).
- **`SourceEnvelopeBuilder`** -- additive availability envelope.

## MLB adapter behavior

Lineup CONFIRMATION via boxscore `battingOrder` presence is the v1 availability signal -- reliable and
free. Player injury inference is intentionally NOT included (no reliable free pregame injury source;
nothing invented).

- both batting orders posted -> **observed** (depth shallow).
- one posted -> **proxy** (depth shallow, "one starting lineup confirmed").
- neither posted -> **missing** ("starting lineups not yet posted", no depth record) -- the common
  pregame state.
- provider could not supply (no gamePk / http failure) -> **unavailable** (no depth record).

## Envelope fields (lineup_injury)

`SignalKey=availability`, `SourceGroup=lineup_injury`, `Sport=mlb`, `Provider=mlb_statsapi` (null when
unavailable), `Status` (observed/proxy/missing/unavailable), `Depth` (shallow when confirmed, else none),
`Observed`, `UpdatedAtUtc` + `FreshnessSummary` (from the boxscore fetch time), `ArtifactSafeSummary`,
`ModelContextSummary`.

## Observed / proxy / missing / unavailable behavior

Mapped above. Statuses reuse the existing `SourceSignalStatuses` vocabulary (no new status). `stale`
remains deferred (freshness is captured, but no threshold engine yet).

## Source-depth behavior

`lineup_injury` is `shallow` when at least one lineup is confirmed, never `enriched` (lineup confirmation
is coarse availability, not deep form). Missing/unavailable emit no depth record, so they do not inflate
depth. Availability is NOT added to `GroundedSignals`/`SignalAvailability`, so `SourceSufficiency` breadth
and confidence are untouched -- availability is observed-only diagnostic depth this slice.

## Fallback behavior

Fail-soft throughout: a missing gamePk, a network/http failure, or an unparseable boxscore returns null ->
the envelope reports `unavailable` and the run proceeds unchanged. A pregame game with no posted lineup
reports `missing` honestly. The run never fails because availability is absent.

## Tests

- `SourceEnvelopeBuilderTests` +4: null->unavailable, both->observed/shallow, one->proxy/shallow,
  none->missing/no-depth.
- `SourceDepthEvaluatorTests` +3: confirmed->shallow lineup_injury, missing->no record, null->no record.
- `SportsRetrieverTests` updated: registered the availability tool/handler in the test gateway (empty
  boxscore -> missing, breadth unchanged). Root-caused 6 transient failures to the unregistered tool;
  fixed by registration only.
- Full `DevCore.Api.Tests` suite: **788 passed / 0 failed** (was 781; +7). No regressions.
- No Python/frontend changes -> no py_compile / sports-app runs needed.

## Verification artifact (live)

Real MLB run `dbb5433e` (2026-06-22): completed; `sourceEnvelopes` carried three groups --
`starting_pitching` observed/enriched, `market_odds` observed/enriched, and **`lineup_injury` proxy/shallow
(mlb_statsapi, "one starting lineup confirmed", fresh 2026-06-22T18:22Z)**. The honest partial state (one
lineup posted) did not fail the run. `sourceDepth` showed `lineup_injury=shallow` (not overstated). lean
away @ 0.8, posture monitor -- unchanged decision logic.

## Cross-sport reuse notes (NBA/NFL)

The `lineup_injury` envelope + status/depth contract is sport-agnostic. Future adapters drop in without
touching the envelope, SourceDepth seam, or the artifact surface:
- **NBAAvailabilityAdapter** -> injury report (active/inactive, probable/questionable/out) -> observed/proxy.
- **NFLAvailabilityAdapter** -> injury report + inactive list + depth-chart -> observed/proxy.
Each emits a `lineup_injury` envelope from its provider; the cross-sport contract is unchanged.

## Deferred items

- Player injury status (no reliable free pregame source) -- lineup confirmation only this slice.
- Analyzer prompt-injection of the availability summary (kept prompt doctrine unchanged; data-block
  injection is a documented follow-up).
- `stale` status (freshness captured; threshold engine deferred).
- NBA/NFL availability adapters.
- Named-risk-grounding integration: with `lineup_injury` now groundable, a follow-up can let availability
  named-risks map to it (the original motivation) -- not wired this slice.

## Recommended next slice

**Schedule/Fatigue Context v1** -- generalize NBA rest + add an MLB bullpen-fatigue proxy
(`rest_travel` / `bullpen_availability` envelopes), the next-largest ungrounded-risk lane. Alternatively
**Market Agreement Projection v1** (read-time lean-vs-market agreement) or **Reconcile Directional-Contrast
Cohort v1** once games 200013-200022 are Final.
