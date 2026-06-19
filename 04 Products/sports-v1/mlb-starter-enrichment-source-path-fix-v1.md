# MLB Starter Enrichment Source-Path Fix v1

**date:** 2026-06-19
**status:** IMPLEMENTED. Starter season quality/form now fetched from the people endpoint; enriched regime triggers on live data.
**classification:** source-path implementation (TDD). Code + tests + live smoke + docs. No generation, no reconciliation, no advisory/enforcement.

**Anchor:** Presence is not depth. The depth contract exists; now the source path actually enriches.

## 1. Problem

MLB Starting Pitching Quality/Form Enrichment v1 read pitcher season stats from a schedule hydrate
(`probablePitcher(stats(group=[pitching],type=[season]))`). The Source Depth Contract v1 live smoke
found that hydrate does NOT attach stats, so in production `MlbPitcherQuality.DataAvailable` was always
false and Source Depth stayed `identity_only`. This slice switches enrichment to the StatsAPI people
endpoint so quality actually populates and Source Depth reports `enriched` when stats exist.

## 2. Failed Schedule-Hydrate Finding

Confirmed live (2026-06-19/20/21): probable pitchers attach to the schedule, but
`probablePitcher.stats` is absent across all games and three hydrate-syntax variants. The schedule
hydrate is not a viable source for pitcher season stats.

## 3. People Endpoint Path

Per probable pitcher: `GET /api/v1/people/{id}/stats?stats=season&group=pitching&season={year}` on the
existing `statsapi.mlb.com` base. `{id}` comes from `schedule.probablePitcher.id`; `{year}` is the
game's calendar year (baseball season == calendar year). The response reuses the same
`stats[].splits[].stat` shape the (unused) hydrate would have produced, so the existing stat parser is
reused. The schedule call reverts to the plain `hydrate=probablePitcher` (id + name + handedness).

## 4. Fields Parsed

From the season pitching split (raw stats-api values preserved, traceable):

| field | stats-api key | type |
|---|---|---|
| Era | `era` | string |
| Whip | `whip` | string |
| StrikeOuts | `strikeOuts` | int |
| Walks | `baseOnBalls` | int |
| InningsPitched | `inningsPitched` | string (workload proxy) |
| GamesStarted | `gamesStarted` | int (workload proxy) |

No injury/return-from-injury, recent-form, or pitch-count claims -- the endpoint's season split does
not provide them, so nothing is invented.

## 5. Fan-Out Decision

Accepted: a scoped **2-call fan-out per MLB game** -- one people call for the home probable starter,
one for the away probable starter -- in addition to the single schedule call. This is the minimum to
enrich both starters. Per-`(pitcher, season)` results are cached (30 min) so a repeated pitcher is not
refetched within a retrieval window; the existing per-matchup 30-min cache already prevents refetch on
the same game. No broader endpoints, no roster/team-wide fan-out.

## 6. Fail-Soft Behavior

Every degradation preserves identity-only starters and never throws:

- missing `probablePitcher.id` -> no people call; quality `DataAvailable=false`, reason "probable pitcher id missing".
- people call network failure -> `DataAvailable=false`, reason "season pitching stats unavailable".
- people call non-2xx -> `DataAvailable=false`, reason "season pitching stats unavailable".
- unparseable body -> `DataAvailable=false`, reason "season pitching stats unparseable".
- no season pitching split -> `DataAvailable=false`, reason "no season pitching stats published".
- one starter's call fails while the other succeeds -> the succeeding starter is enriched; the failing
  one stays identity-only. Retrieval still returns the full `MlbStarterContext`.

## 7. SourceDepth Impact

When either starter's quality is `DataAvailable=true`, `SourceDepthEvaluator` (unchanged) returns
`starting_pitching = enriched`; otherwise `identity_only`. So this slice flips the live MLB regime from
identity_only to enriched whenever StatsAPI publishes season stats. Verified by the retriever test
(`retrieve_threads_enriched_source_depth_for_mlb_without_changing_breadth`) and the live smoke.

## 8. SourceSufficiency Non-Impact

No change to `SourceSufficiencyBuilder` / `SourceSignalTaxonomy`. Quality remains nested inside the
already-grounding `starting_pitching`; it adds no grounded signal and no source group. A starter+market
MLB run still grounds `[starting_pitching, market]` and bands `moderate` whether identity_only or
enriched. No new source group. No confidence/lean/posture/threshold/buyer change. No FastAPI change --
the wire contract (`MlbStarterContext` with nested `MlbPitcherQuality`) is unchanged; only where the
.NET client sources the values changed.

## 9. Tests / Live Smoke

TDD red->green. `DevCore.Api.Tests` 747 -> 751 (+4 new client tests; 2 existing enriched retriever
tests updated to the people path). No FastAPI change -> pytest unchanged (115).

- `MlbStarterClientTests` (new, url-routing fake handler):
  - people-endpoint season stats -> `MlbPitcherQuality.DataAvailable=true` with parsed fields.
  - both starters get quality from their own pitcher id (per-id routing).
  - one failed stats call (away 500) does not fail retrieval; home enriched, away identity-only.
  - missing probable pitcher id -> identity-only, no people call made.
- `SportsRetrieverTests`: enriched depth + breadth-unchanged now exercised via the people path.
- `SportsAnalysisRequestSerializationTests` (unchanged): the analyzer request still carries enriched
  quality (contract untouched).

**Live smoke (no-spend, no-model, 2026-06-21):** real schedule (plain `probablePitcher`) -> pitcher
ids; people endpoint returned season stats for both starters of a real game (Gerrit Cole ERA 2.57,
WHIP 1.00; Chase Burns ERA 2.01, WHIP 1.02). Both `DataAvailable=true` -> `SourceDepthEvaluator`
returns `starting_pitching = enriched`. Confirms the production path. `git diff --check` clean; no
migration, no frontend, no model call.

## 10. Deferred Items

- Depth-aware SourceSufficiency band (let the band reflect depth) -- still deferred.
- A budgeted starter-depth MLB generation + reconcile to read the enriched regime against outcomes.
- Recent-form / last-X splits, pitch-count, injury/availability -- not provided by the season split.
- People-endpoint resilience hardening (retry/backoff, partial-season edge cases) if volume grows.
- team_form / lineup / bullpen / moneyline -- unchanged, deferred.
- Advisory/enforcement, Probe, confidence/posture/threshold changes -- unchanged, deferred.
