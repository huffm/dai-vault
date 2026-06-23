# Decision Freshness Read Model v1

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code only (no schema/migration). Read-only, baseline-only.
No provider calls, refresh, regeneration, prompt, buyer-UI, confidence/lean/posture/reconciliation change;
not full CLV.
**implements:** the recommended next slice from `decision-freshness-architecture-v1.md`. First piece of the
Decision Freshness layer.

## What shipped

A single read-only `DecisionFreshnessResult` that combines a run's persisted baseline (lean, generated
time, SourceEnvelopes from OutputJson) with Close-Market Confirmation Read v1, producing an overall
freshness status + buyer-safe summary. No new data is fetched; refreshed deltas are a later slice.

- **`DecisionFreshnessCalculator`** (pure): maps the close-market confirmation result + baseline source
  envelopes to overall status, buyer-safe summary, market-support summary, line-quality summary,
  source-update summary, and per-lane source deltas.
- **`DecisionFreshnessResult` / `DecisionFreshnessSourceDelta`** (read DTOs).
- **`DecisionFreshnessService`**: loads the run (lean, StartedUtc, OutputJson), deserializes the baseline
  SourceEnvelopes fail-soft, calls `ICloseMarketConfirmationService`, builds the result. Read-only.
- **Dev-gated endpoint** `GET /api/agent-runs/{agentRunId}/decision-freshness` (tenant-scoped, like
  close-market-confirmation). Internal -- not a buyer surface.

## Status model

`fresh_supported`, `fresh_neutral`, `weakened`, `stale` (reserved for the refresh slice; v1 never emits
it), `unavailable`, `not_comparable`, `insufficient_history`. Diagnostic only -- never a decision mutation.

## Close-Market Confirmation mapping

- observed + moved_toward_lean -> `fresh_supported`
- observed + moved_against_lean -> `weakened`
- observed + neutral -> `fresh_neutral`
- no_near_close_snapshot -> `insufficient_history`
- no_artifact_lean / not_comparable -> `not_comparable`
- no_generation_snapshot / unavailable -> `unavailable`

Market support reuses the close-market buyer-safe phrase directly ("Market moved toward this read." /
"Market moved against this read." / "Market stayed mostly neutral." / "No close-market comparison
available yet.").

## Baseline source-envelope summary behavior

v1 summarizes the BASELINE source envelopes only -- it does NOT compare refreshed envelopes yet. Each
persisted envelope becomes a `DecisionFreshnessSourceDelta` with the baseline status/depth and a
conservative classification: `unchanged` for present lanes (observed/proxy), `unavailable` for missing/
unavailable/not_attempted, `not_applicable` where applicable. A derived `market_movement` delta comes from
the close-market state (`unchanged` when observed, `insufficient_history` when no near_close). No lane is
ever classified `weakened` in v1 (that requires a refreshed comparison).

## Line quality (v1, conservative)

Flags degradation only: observed + moved_toward_lean (lean-side number got more expensive) -> "Earlier
number was better."; otherwise "Current number is similar."; no observed comparison -> "Line comparison
unavailable." A precise better/worse split is deferred (and never exposed as a CLV claim).

## Buyer-safe language (used)

Overall: "This read has market support and source context is available." / "Market stayed mostly neutral
since generation." / "This read has weakened since generation." / "Freshness is limited because no
close-market comparison is available yet." / "Freshness is not comparable for this read yet." / "Freshness
unavailable." Source update: "Source context is available for: market, starting pitching, lineup, bullpen."
Forbidden terms (asserted absent in tests): CLV, sharp, lock, basis point, implied probability, guaranteed,
event id.

## CLV boundary

`TrueClvEvaluable` stays false (carried from close-market confirmation): the artifact has a directional
lean, not a formal market selection (type/side/line/price). No CLV claim.

## Tests

- `DecisionFreshnessTests`: pure calculator (toward->fresh_supported, against->weakened, neutral->
  fresh_neutral, no_near_close->insufficient_history, no_artifact_lean->not_comparable, unavailable->
  unavailable, baseline lanes summarized, buyer-safe summaries avoid jargon + CLV false) + service
  (fresh_supported from persisted baseline + close, unavailable for missing run). 10 tests.
- Existing close-market / market-memory / reconciliation tests preserved.
- Full `DevCore.Api.Tests` suite: **871 passed / 0 failed** (was 861; +10). No regressions.
- No Python/frontend change.

## Verification

Live: `GET /api/agent-runs/e5b5433e/decision-freshness` -> `overallStatus=fresh_neutral`, lean away,
trueClvEvaluable=false; marketSupport "Market stayed mostly neutral.", lineQuality "Current number is
similar.", sourceUpdate "Source context is available for: bullpen, lineup, market, starting pitching.",
buyerSafe "Market stayed mostly neutral since generation."; close-market confirmation embedded
(observed/neutral); five source deltas (market_odds observed/enriched, starting_pitching observed/enriched,
lineup_injury observed/shallow, bullpen_availability proxy/shallow, market_movement observed) all
`unchanged`. No provider call; no artifact/buyer/prompt/confidence/posture/reconciliation change.

## What did not change

No provider calls; no refresh/rerun/regeneration; no scheduler/polling; no prompt, buyer UI/copy, artifact
JSON, confidence, lean, posture, or reconciliation change. Read-only and additive (new model/service + one
GET endpoint + DI).

## Follow-ups

- **Source Delta Calculator v1** -- compare baseline envelopes to REFRESHED envelopes (real
  strengthened/weakened/newly_available deltas).
- **Refresh Trigger v1** -- manual source refresh (reuse near_close), no regeneration.
- **Decision Freshness Artifact Projection v1** -- internal artifact DTO derive-on-read.
- **Buyer-Safe Freshness Copy v1**; **Freshness Calibration v1** (statuses vs outcomes).

## Recommended next slice

**Source Delta Calculator v1** -- the next leg: a pure baseline-vs-refreshed envelope comparison producing
real strengthened/weakened/newly_available deltas (the part v1 stubs as `unchanged`). Alternatively
**Refresh Trigger v1** (which produces the refreshed envelopes to compare against) or **Decision Freshness
Artifact Projection v1**.
