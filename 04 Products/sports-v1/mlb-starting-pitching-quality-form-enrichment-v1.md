# MLB Starting Pitching Quality/Form Enrichment v1

**date:** 2026-06-19
**status:** IMPLEMENTED. Source-depth enrichment; observed analyzer input change for future MLB runs.
**classification:** source implementation (TDD). Code + tests + docs. No generation, no reconciliation, no advisory/enforcement.

**Anchor:** Presence is not depth. Probable starter identity is not pitcher quality.

## 1. Problem

Market-Aware MLB Moderate Calibration Review v1 found that all 8 market-aware moderate runs carried
an identical observed projection (band/PF/confidence/posture) and the readiness signal had zero
discriminating power. The recurring root was source DEPTH: `starting_pitching` was grounded at
probable-starter *identity* only (name + handedness); `market` was the run line only. The model
named the right risks but had no grounded depth to weigh against the market read.

This slice adds the first depth layer: when StatsAPI publishes season pitching stats for a probable
starter, enrich the analyzer input with that quality/form context. It is grounded as DEPTH inside the
existing `starting_pitching` signal -- not a new source group, and (this slice) not a band-rule change.

## 2. Source Path Used

MLB StatsAPI, the existing MLB source path -- no new provider, no new tool, no extra fan-out.

- `MlbStarterClient` (`dai/platform/dotnet/DevCore.Api/Sports/MlbStarterClient.cs`) already fetches
  `/api/v1/schedule?sportId=1&date={date}&hydrate=probablePitcher`. The hydrate was expanded to
  `hydrate=probablePitcher(stats(group=[pitching],type=[season]))` so the season pitching split rides
  along the same single schedule call. No second request per pitcher.
- Flows through the existing seam: `MlbStarterClient` -> Tool Gateway (`pitching.mlb.probable_starters`)
  -> `SportsRetriever` MLB branch -> `SportsRetrievalOutput.MlbStarterContext` -> `SportsAnalyzer`
  builds `SportsAnalysisRequest.MlbStarterContext` -> FastAPI `analyze_mlb`. Because quality is nested
  inside `MlbStarterContext`, no new threading wiring was needed in `SportsAnalyzer`.

## 3. Fields Added

New typed record `MlbPitcherQuality` (`DevCore.AiClient/MlbStarterContext.cs`), one per starter, added
as optional `HomeStarterQuality` / `AwayStarterQuality` on `MlbStarterContext` (additive -- identity-only
construction still compiles):

| field | type | source (statsapi season pitching) | notes |
|---|---|---|---|
| `DataAvailable` | bool | derived | false when no season split published |
| `MissingReason` | string? | derived | e.g. "no season pitching stats published"; null when available |
| `Era` | string? | `era` | raw stats-api string, e.g. "3.45" |
| `Whip` | string? | `whip` | raw string, e.g. "1.21" |
| `StrikeOuts` | int? | `strikeOuts` | season K |
| `Walks` | int? | `baseOnBalls` | season BB |
| `InningsPitched` | string? | `inningsPitched` | raw string "84.1"; workload proxy |
| `GamesStarted` | int? | `gamesStarted` | workload proxy |

Raw source values are preserved as-is (era/whip/innings are strings in StatsAPI; counts are ints) so
they stay traceable for artifact/debug review. The pydantic mirror `MlbPitcherQuality` +
`homeStarterQuality`/`awayStarterQuality` was added in `app/models/sports.py`.

Deliberately NOT added (source does not reliably support them via this single-call path; not invented):
recent-starts/last-X-games splits, pitch-count, and return-from-injury/availability status. Season IP +
GS serve as the workload proxy. These remain candidates for a later slice.

## 4. Fail-Soft Behavior

- Stats hydrate absent/empty, or no `pitching`+`season` split present -> `MlbPitcherQuality(DataAvailable=false,
  MissingReason="no season pitching stats published", all stats null)`. The starter stays usable at
  identity-only depth (name + handedness preserved); `starting_pitching` still grounds.
- Any individual stat field missing -> that field stays null; the rest are emitted.
- Network/HTTP failure or starters unannounced -> unchanged from before: `MlbStarterContext` null,
  `starting_pitching` ungrounded. Quality enrichment never breaks artifact generation.
- The FastAPI prompt emits only the fields actually present; nothing is fabricated.

## 5. Analyzer Input Change

`analyze_mlb` (`app/services/sports_analyzer.py`) now injects per-starter season form into the
`[starter data]` block when quality is available, via `_format_pitcher_quality(...)`, e.g.:

```
[starter data]
home starter: Gerrit Cole (RHP)
home starter season form: ERA 3.45, WHIP 1.21, 95 K, 22 BB, 84.1 IP, 14 GS
away starter: Jose Berrios (RHP)
away starter season form: ERA 4.10, WHIP 1.30, 80 K, 30 BB, 78.0 IP, 13 GS
use these starters in the starting pitching factor. note handedness advantage vs lineup if relevant.
use the provided season stats (era, whip, k, bb, innings, games started) as pitcher quality/form. do not fabricate any stats beyond those provided.
```

When no quality is available, the block is unchanged from before and keeps the original instruction
("only name and handedness are available. do not fabricate era or recent form"). This is a NEW
source-depth regime: future MLB runs with season stats see richer starter context, so their lean /
confidence may differ from the identity-only cohort -- treat as a regime change, not a like-for-like
comparison.

## 6. SourceSufficiency Non-Change

No change to `SourceSufficiencyBuilder` / `SourceSignalTaxonomy` this slice. The band is derived from
`GroundedSignals.Count` (`DeriveBand`), and quality is nested inside the already-grounding
`MlbStarterContext`; it adds no new grounded signal and no new source group. A starter+market MLB run
still grounds exactly `[starting_pitching, market]` and still bands `moderate`. Depth is now present in
the evidence but is NOT yet reflected in the sufficiency band -- that is the Source Depth Contract's job
(deferred), per the calibration review's "contract before content" sequencing.

## 7. New Regime Note

This changes future MLB analyzer input. The market-aware moderate cohort (AgentRunKey 180013-180020)
was generated identity-only; runs generated after this slice with season stats are a distinct
starter-depth regime. Do not pool them with the identity-only cohort when reading calibration, and do
not compare across the change as if only coverage widened.

## 8. Tests

TDD, red->green throughout.

.NET (`DevCore.Api.Tests`, 735 -> 738, +3):
- `SportsRetrieverTests.retrieve_grounds_mlb_starter_quality_when_season_stats_present` -- season stats
  parsed into both starters' quality (ERA/WHIP/K/BB/IP/GS); `GroundedSignals` unchanged (`starting_pitching`).
- `SportsRetrieverTests.retrieve_mlb_starter_quality_is_failsoft_when_season_stats_absent` -- starters
  announced, no stats -> identity preserved, quality `DataAvailable=false` + reason, `starting_pitching` grounds.
- `SportsAnalysisRequestSerializationTests.request_serializes_mlb_pitcher_quality_so_it_reaches_fastapi`
  -- the wire contract serializes quality (camelCase web defaults) so it reaches the analyzer request.

FastAPI (`pytest`, 113 -> 115, +2):
- `test_analyze_mlb_includes_pitcher_quality_when_present` -- quality reaches the prompt ("season form",
  "ERA 3.45", "95 K", "use the provided season stats").
- `test_analyze_mlb_omits_pitcher_quality_when_absent` -- identity-only keeps the no-stats instruction.

Full suites green: `DevCore.Api.Tests` 738/738; FastAPI 115/115. No model call (tests mock the model).

## 9. Deferred Items

- Source Depth Contract v1 (make depth inspectable + let the band/projection reflect it) -- the
  prerequisite this slice intentionally did not pre-empt.
- Recent-starts / last-X-games form, pitch-count proxy, return-from-injury/availability (no reliable
  single-call path; not invented here).
- `team_form`, `lineup_injury`, bullpen, weather, moneyline/h2h -- all unchanged, deferred.
- Advisory/enforcement, Probe execution, buyer display, confidence/posture/threshold changes,
  calibration decisions, WNBA -- all unchanged, deferred.
- A budgeted generation to capture a starter-depth MLB cohort and reconcile it (separate slice).
