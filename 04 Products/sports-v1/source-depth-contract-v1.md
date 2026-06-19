# Source Depth Contract v1

**date:** 2026-06-19
**status:** IMPLEMENTED. Observed-only source-depth representation; MLB starting_pitching covered. Recommendation behavior unchanged.
**classification:** platform contract + niche coverage (TDD). Code + tests + docs. No generation, no reconciliation, no advisory/enforcement.

**Anchor:** Breadth answers "how many independent source groups grounded?" Depth answers "how informative was a grounded group?" Probable-starter identity is not pitcher quality.

## 1. Problem

MLB Starting Pitching Quality/Form Enrichment v1 added optional season pitching quality/form under
`MlbStarterContext`, but that depth is invisible to `SourceSufficiency`: it nests inside the existing
`starting_pitching` group, so it adds no grounded signal and the band stays `moderate` whether the
starter is identity-only or enriched. The Market-Aware MLB Moderate Calibration Review showed the
projection cannot distinguish shallow from deep grounding. This slice makes source DEPTH explicit,
inspectable, and persisted on the artifact -- without changing any recommendation behavior.

## 2. Source Depth Contract Shape

New pure types (`DevCore.Api/AgentRuns/SourceDepth.cs`):

```
SourceDepthLevels: none | identity_only | shallow | enriched

SourceDepthRecord(
  Group:         string   // canonical source group, e.g. "starting_pitching", "market_odds"
  Level:         string   // one of SourceDepthLevels
  Observed:      bool     // true = depth read from source data; false = inferred from absence
  Detail:        string?  // optional observed detail
  MissingReason: string?  // optional reason when below enriched / unavailable
)
```

`SourceDepthEvaluator.Evaluate(competition, MlbStarterContext?, BaseballMarketContext?)` derives the
records deterministically. MLB only this slice; other competitions return `[]` (contract exists,
niche coverage added later). Breadth (`SourceSufficiency`) is untouched -- the two are independent.

## 3. MLB Behavior (this slice)

| situation | starting_pitching depth | Observed | notes |
|---|---|---|---|
| starters announced, `MlbPitcherQuality.DataAvailable=true` (either side) | `enriched` | true | detail names the season stats |
| starters announced, no published season stats (DataAvailable=false) | `identity_only` | true | MissingReason carried; identity preserved |
| identity-only context (legacy/no quality) | `identity_only` | true | -- |
| starters not announced (`MlbStarterContext` null) | `none` | false | inferred from absence |
| grounded run line (`BaseballMarketContext` present) | `market_odds` = `shallow` | true | run line only, no moneyline/consensus depth |

`Observed` cleanly separates "depth seen in data" (identity_only/shallow/enriched) from "inferred
from absence" (none).

## 4. Where It Threads

`SourceDepthEvaluator` (retrieve) -> `SportsRetrievalOutput.SourceDepth` -> `SportsComposer` persists
it onto `AgentRunExecutionResult.SourceDepth` (success AND analyze-failure paths) -> projected
read-only on `AgentRunArtifactDto.SourceDepth` via `GET /api/agent-runs/{id}/artifact`. Persisting it
(not recomputing at read time) is deliberate: the depth label must survive on the stored artifact so
historical runs can be segmented by source regime. No FastAPI change -- depth is .NET-derived from the
retrieved contexts; the analyzer already receives the quality fields and does not need the label.

## 5. SourceSufficiency Non-Change (verified)

No change to `SourceSufficiencyBuilder` / `SourceSignalTaxonomy`. `DeriveBand` still counts grounded
signals; depth nests inside the already-grounding `starting_pitching` and adds no new grounded signal
and no new source group. A starter+market MLB run still grounds `[starting_pitching, market]` and
bands `moderate` whether starter depth is identity_only or enriched. Confidence, lean, posture,
thresholds, and buyer copy are untouched. The retriever test asserts breadth is unchanged while depth
is populated.

## 6. Live StatsAPI Smoke (no-spend, no-model)

Read-only smoke against real StatsAPI (`2026-06-19/20/21`):

- Probable pitchers are present (14/14 games) but **the schedule hydrate
  `probablePitcher(stats(group=[pitching],type=[season]))` does NOT attach season stats inline** --
  0/14 games carried `probablePitcher.stats`, across three hydrate-syntax variants. This is the exact
  hydrate string shipped by MLB Starting Pitching Quality/Form Enrichment v1.
- Season stats DO exist and are reachable via the **people endpoint**
  `/api/v1/people/{id}/stats?stats=season&group=pitching&season=2026` (confirmed for Will Warren
  ERA 3.47, Andrew Abbott ERA 3.95, Troy Melton ERA 2.81).

Consequence: in production today, `MlbPitcherQuality.DataAvailable` is false, so Source Depth
correctly reports `identity_only` -- the fail-soft path is validated on live data. The `enriched`
regime is achievable but requires correcting the enrichment retrieval to call the people endpoint per
probable pitcher (a deliberate 2-call fan-out), which is a follow-up to the enrichment slice and is
OUT of this contract slice's boundary (no new providers/fan-out here). Recorded as an open risk.

## 7. Tests

TDD red->green. `DevCore.Api.Tests` 738 -> 747 (+9). No FastAPI change -> no pytest change.

- `SourceDepthEvaluatorTests` (6): identity_only, enriched, fail-soft identity_only+reason,
  none-inferred-from-absence, shallow market_odds, non-MLB returns no records.
- `SportsRetrieverTests.retrieve_threads_enriched_source_depth_for_mlb_without_changing_breadth`:
  depth threads through retrieve AND breadth (`GroundedSignals`) is unchanged.
- `SportsComposerTests.compose_persists_source_depth_from_retrieval`: depth lands on the persisted
  `AgentRunExecutionResult` and survives the read-path JSON contract.
- `AgentRunsControllerTests.artifact_endpoint_projects_source_depth_without_mutating_confidence`:
  `GET /artifact` surfaces depth and leaves confidence unchanged (0.68 in == 0.68 out).

`git diff --check` clean. No migration, no frontend, no model call.

## 8. Deferred Items

- **MLB Starter Enrichment Source-Path Fix v1** (recommended next): switch enrichment retrieval to the
  people-endpoint season-stats call per probable pitcher (deliberate 2-call fan-out) so the `enriched`
  regime actually triggers. The schedule-nested hydrate does not deliver stats.
- Making the SourceSufficiency band/projection *reflect* depth (a depth-aware band) -- intentionally
  not done; this slice only represents depth.
- Buyer-visible depth wording -- deferred; the dev/artifact projection comes first.
- Depth coverage for non-MLB niches, market moneyline/consensus depth, team_form/lineup depth.
- Advisory/enforcement, Probe, confidence/posture/threshold changes -- all unchanged, deferred.

## 9. New Regime Note

Once enrichment delivers stats, MLB runs split into identity-only vs enriched source regimes, now
machine-readable on the artifact (`SourceDepth`). Do not pool enriched artifacts with the earlier
identity-only calibration cohort.
