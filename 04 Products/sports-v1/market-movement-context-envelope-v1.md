# MarketMovementContext Envelope v1

**date:** 2026-06-22
**status:** IMPLEMENTED + verified (TDD). Code only (no schema/migration). Observed-only. No provider
calls, no close capture, no write-path/artifact-generation/prompt/buyer/confidence/posture/reconciliation
change.
**implements:** the recommended next slice from `market-movement-derivation-v1.md`. Turns the read-only
movement derivation into a source envelope for the `market_movement` lane.

## What shipped

An observed-only `market_movement` source envelope built from the existing movement derivation.

- **`MarketMovementEnvelope`** (typed DTO): richer than the generic `SourceSignalEnvelope` -- it carries
  batch count, first/latest observed time, close-observed flag, consensus first/latest + flip, line
  movement count, and a capped deterministic sample of representative line movements, plus the standard
  source-lane fields (signalKey/sourceGroup=market_movement, sport, provider, status, observed, summaries).
- **`MarketMovementStatuses`** (lane-scoped): `observed` (>=2 batches), `insufficient_history` (1 batch),
  `unavailable` (0 batches). Scoped to this lane -- does NOT extend the shared SourceSignalStatuses.
- **`MarketMovementEnvelopeBuilder`** (pure): MarketMovement? -> MarketMovementEnvelope.
- **`IMarketMovementEnvelopeService` / `MarketMovementEnvelopeService`** (thin adapter): derives via
  `IMarketMovementDerivationService`, then builds the envelope. Registered in DI. No HTTP endpoint
  (internal -- consistent with the Query and Derivation services).
- **`MarketMovement`** gained additive `Provider` + `ProviderEventId` (from the latest batch) so the
  envelope can label provenance. No change to movement-delta semantics.

## Status / observed semantics

- 0 batches -> `unavailable`, Observed=false, no movement claimed.
- 1 batch -> `insufficient_history`, Observed=false, BatchCount=1, no line movements (never fabricated).
- >=2 batches -> `observed`, Observed=true, line movement count + representative sample (capped at 5,
  deterministic order from the calculator).
- `closeObserved` is carried straight from the derivation -- true only when a `near_close` capture exists
  (none today). latestObserved is never presented as close. Language stays "observed market movement,"
  never "closing line value."

## Surfacing (deferred, deliberate)

The envelope is an internal service, not wired into the artifact JSON or any buyer surface this slice --
matching the established market-memory pattern (Query v1 and Derivation v1 added internal services without
surfacing). Buyer surfaces are unchanged. A future slice can surface it as a derive-on-read field on the
internal `/artifact` DTO when there is enough multi-batch history for it to be meaningful.

## Boundaries (not changed)

No provider calls; no scheduled polling; no closing-snapshot capture; no change to the ledger write path,
movement derivation semantics, artifact generation, prompts, buyer copy/UI, lean/confidence/posture, or
reconciliation. No frontend.

## Tests

- `MarketMovementEnvelopeTests` (+10): builder -> no-history=unavailable; single-batch=insufficient_history
  (no movement); two-batch=observed; consensus flip carried into envelope + summary; close not claimed
  without `near_close` (and claimed with it); representative line movements capped at 5 + deterministic;
  marketType passed through. Service (in-memory, real derivation+query path) -> unavailable / insufficient
  / observed.
- Existing `MarketSnapshotLedgerTests`, `MarketSnapshotQueryServiceTests`, `MarketMovementCalculatorTests`,
  `MarketMovementDerivationServiceTests` preserved.
- Full `DevCore.Api.Tests` suite: **830 passed / 0 failed** (was 820; +10). No regressions.
- No Python/frontend change.

## Verification

- Builder + service fully covered by deterministic tests.
- Live DB cross-check: ExternalGameId 824261 (run `e2b5433e`) has exactly one persisted batch and zero
  `near_close` batches -- so the envelope service reports `insufficient_history`, BatchCount 1, no line
  movements, closeObserved false (no movement/close overclaim). Derivation is a pure DB read; no provider
  call occurs.

## Recommended next slice

**Closing Snapshot Capture v1** -- a near-start capture mode that emits the `near_close` capture reason,
which would make `closeObserved` real and unlock artifact-vs-close. Alternatively surface this envelope on
the internal `/artifact` read DTO, or **Reconcile Directional-Contrast Cohort v1** once games 200013-200022
are Final.
