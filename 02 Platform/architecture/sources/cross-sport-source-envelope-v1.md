# Cross-Sport Source Envelope v1

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code change (dai). Backward compatible. No confidence/
posture/lean logic, prompt-doctrine, buyer-UI, reconciliation, or migration change. No new providers.
**implements:** the recommended immediate next slice from
`02 Platform/architecture/sources/cross-sport-source-component-architecture-v1.md`.

## What was implemented

Introduced a reusable, normalized `SourceSignalEnvelope` and retrofitted the two already-grounded MLB
signals (market + starter depth) onto it -- no new providers, no new sports. The envelope is the reusable
contract (meaning); existing clients remain the adapters (provider detail). It is observed-only and
additive end to end.

- **`SourceSignalEnvelope`** (`DevCore.Api/AgentRuns/SourceSignalEnvelope.cs`, new) -- normalized per-signal
  view: SignalKey, SourceGroup (canonical `SourceGroups`), Sport, Provider, Status, Depth, Observed,
  UpdatedAtUtc, SourceRef, FreshnessSummary, ArtifactSafeSummary, ModelContextSummary.
- **`SourceSignalStatuses`** -- observed / proxy / missing / unavailable / not_attempted / not_applicable.
  Aligns with the existing availability vocabulary (`grounded` generalizes to `observed`). **`stale` was
  intentionally NOT added to runtime** -- freshness is modeled (UpdatedAtUtc + FreshnessSummary), but a
  freshness-derived `stale` status is deferred until thresholds exist (per the architecture plan).
- **`SourceEnvelopeBuilder`** (pure) -- wraps the grounded contexts using the authoritative
  `SourceDepthRecord[]` for the depth level, so envelopes and SourceDepth **cannot diverge**. MLB-scoped
  (matches `SourceDepthEvaluator`); other competitions return `[]`.
- **Wiring (additive, mirrors `SourceDepth`):** `SportsRetriever` builds envelopes from the same inputs +
  depth and passes them to `SportsRetrievalOutput.SourceEnvelopes`; `SportsComposer` persists them on
  `AgentRunExecutionResult.SourceEnvelopes` (success and failed paths); the artifact DTO + the
  `GET /api/agent-runs/{id}/artifact` projection surface them. All new fields are trailing/nullable.

## Envelope fields

`SignalKey`, `SourceGroup`, `Sport`, `Provider?`, `Status`, `Depth`, `Observed`, `UpdatedAtUtc?`,
`SourceRef?`, `FreshnessSummary?`, `ArtifactSafeSummary?`, `ModelContextSummary?`.

## Mapped existing signals

- **market_odds** (`market`) -- provider `odds_api`. Status observed when grounded, missing when absent.
  Depth = the authoritative market_odds level: `enriched` (multi-book moneyline consensus, from Market Odds
  Depth v1) or `shallow` (single-book run line). Freshness from the bookmaker `UpdatedAt`. ModelContext
  carries the run line plus, when enriched, book count + consensus side.
- **starting_pitching** -- provider `mlb_statsapi`. Status observed when announced, missing when not.
  Depth = the authoritative starting_pitching level: `enriched` (season stats present), `identity_only`
  (announced, no stats), `none` (not announced -> missing envelope). ModelContext carries starter names +
  handedness. (Starter context exposes no per-fetch timestamp yet -> UpdatedAtUtc null.)

## Compatibility behavior (what did not change)

- `SourceDepth` remains the depth authority and its output is byte-for-byte unchanged -- the envelope reads
  it, never overrides it. All prior SourceDepth/market-depth/retrieval/composer/DTO tests stay green.
- No change to lean, confidence, posture, evaluator, reconciliation, buyer UI, or prompts.
- New fields are additive + nullable; older persisted artifacts deserialize fine (envelopes null/empty).
- Analyzer prompt-injection was NOT changed this slice (the model still receives the existing sport-
  specific market/starter context). Routing envelope summaries into model context is deferred.

## Tests

- New `SourceEnvelopeBuilderTests` (8): non-mlb -> empty; market enriched/shallow/missing; starter
  enriched/identity_only/missing; envelope depth always equals the SourceDepthEvaluator level.
- Full `DevCore.Api.Tests` suite: **781 passed / 0 failed** (was 773; +8). No regressions -> backward
  compatibility confirmed.

## Verification artifact (live)

Real MLB run `dab5433e` (2026-06-22): completed; `sourceEnvelopes` present for both groups --
`starting_pitching` observed/enriched/mlb_statsapi and `market_odds` observed/enriched/odds_api
(updatedAt 2026-06-22T18:01Z); `sourceDepth` unchanged (both enriched). lean away @ 0.8, posture monitor
(unchanged decision logic). Confirms retrieval -> compose persist -> DTO surface round-trip.

## Future adapters enabled

The envelope is now the plug for new categories without bespoke branches: AvailabilityContext
(`lineup_injury`), Schedule/Fatigue (`rest_travel` / `bullpen_availability`), Venue/Environment
(`weather_park`). Each is a sport-specific adapter that emits a `SourceSignalEnvelope`; SourceDepth and the
artifact surface already accept them.

## Deferred items

- `stale` status (needs freshness thresholds).
- Analyzer prompt-injection of envelope summaries (data-only block) -- deferred to keep behavior steady.
- Buyer surface of envelopes (artifact DTO carries them; the buyer template is untouched).
- New providers/sports (Availability/Fatigue/Venue adapters), and any gRPC/protobuf promotion -- not now.

## Recommended next slice

**Availability Context v1 (MLB first)** -- the biggest named-risk-grounding win: an MLBAvailabilityAdapter
emitting a `lineup_injury` envelope (confirmed/probable lineup + player availability), now a drop-in given
this envelope contract. Alternatively **Market Agreement Projection v1** (read-time lean-vs-market
agreement from the persisted multi-book consensus) if a quick calibration dimension is preferred, or
**Reconcile Directional-Contrast Cohort v1** once games 200013-200022 are Final.
