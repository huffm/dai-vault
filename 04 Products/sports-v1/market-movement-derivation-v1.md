# Market Movement Derivation v1

**date:** 2026-06-22
**status:** IMPLEMENTED + verified (TDD). Code only (no schema/migration). Read-only derivation. No
provider calls, no close capture, no write-path/artifact/prompt/buyer/confidence/posture/reconciliation
change.
**implements:** the recommended next slice from `market-snapshot-query-v1.md`. Consumes the Query v1
surface; produces movement DTOs sized for MarketMovementContext Envelope v1.

## What shipped

A read-only derivation layer that computes OBSERVED market movement for a game from its persisted
snapshot batches.

- **`MarketMovementCalculator`** (pure, no i/o): given the batch views for one game, selects
  first/latest observed by FetchedAtUtc and derives consensus + per-book line movement.
- **`MarketMovement` / `MarketLineMovement`** (explicit DTOs): game-level movement (batch count,
  first/latest observed time, close-observed flag, consensus first/latest + flip) and per-book line
  movement (first/latest price + priceDelta; first/latest point + pointDelta).
- **`IMarketMovementDerivationService` / `MarketMovementDerivationService`**: reads batches through
  `IMarketSnapshotQueryService` (by ExternalGameId + optional MarketType), then applies the calculator.
  Registered in DI. No HTTP endpoint (internal; no route pattern required one).
- **`MarketCaptureReasons.NearClose`** constant added (reserved) so close detection is honest now and
  ready for a future Closing Snapshot Capture slice. No writer emits it.

## Derivation semantics

- **firstObserved** = earliest persisted batch for the game/market; **latestObserved** = latest. Selected
  by FetchedAtUtc then batch key (deterministic regardless of input order).
- **closeObserved** = true only when a persisted batch carries the `near_close` capture reason. **false
  today** (no close capture exists). latestObserved is never treated as close.
- **Consensus movement:** consensusSideFirst / consensusSideLatest from the batch summaries; **consensusFlip**
  = true when both present and different, false when both present and same, null when either is missing.
- **Per-book line movement:** only for (bookmaker, marketType, side) pairs present in BOTH first and
  latest. spreads/totals compare point movement; h2h compares price movement only when prices exist
  (else no price delta -- never guessed). Spread/total prices are not persisted yet, so those carry only
  point deltas. Unpaired lines are omitted -- no fake deltas.
- **Insufficient data is boring:** zero batches -> null; one batch -> firstObserved only, empty line
  movements (no delta).

## Boundaries (not changed / not done)

No provider calls; no scheduled polling; no closing-snapshot capture; no change to the ledger write path,
artifact generation, prompts, buyer surface/copy, lean/confidence/posture, or reconciliation. No frontend.
No HTTP endpoint. Terminology stays precise: observed movement, not closing movement.

## Tests

- `MarketMovementCalculatorTests` (+10, pure): empty -> null; single batch -> no delta; first/latest by
  FetchedAtUtc regardless of input order; consensus flip (both/same/missing); h2h price movement; spread
  point movement; totals point movement; marketType filtering; unpaired bookmaker omitted; closeObserved
  false without `near_close` and true with it.
- `MarketMovementDerivationServiceTests` (+3, in-memory): derives h2h movement across two persisted
  batches; no history -> null; single batch -> no delta.
- Existing `MarketSnapshotLedgerTests` and `MarketSnapshotQueryServiceTests` preserved.
- Full `DevCore.Api.Tests` suite: **820 passed / 0 failed** (was 807; +13). No regressions.
- No Python/frontend change.

## Verification

- Calculator + service fully covered by deterministic tests.
- Live DB cross-check: ExternalGameId 824261 (run `e2b5433e`) has exactly one persisted batch and zero
  `near_close` batches -- so the service reports BatchCount 1, no line-movement delta, closeObserved
  false (the documented single-batch behavior). Derivation is a pure DB read; no provider call occurs.

## Recommended next slice

**MarketMovementContext Envelope v1** -- emit a `market_movement` `SourceSignalEnvelope` from this
derivation (observed-only, carrying snapshotBatch range + consensus flip), so movement becomes a surfaced
source lane. Alternatively **Closing Snapshot Capture v1** (which would make closeObserved real) or
**Reconcile Directional-Contrast Cohort v1** once games 200013-200022 are Final.
