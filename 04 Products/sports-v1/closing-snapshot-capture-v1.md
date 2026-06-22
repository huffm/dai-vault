# Closing Snapshot Capture v1

**date:** 2026-06-22
**status:** IMPLEMENTED + verified (TDD). Code only (no schema/migration). Pregame-gated capture. No
artifact/buyer/prompt/confidence/posture/reconciliation change; no CLV/beat-close scoring.
**implements:** the recommended next slice from `market-movement-context-envelope-v1.md`. Makes
`closeObserved` real for MarketMovementContext.

## What shipped

A minimal internal capture path that writes a normalized snapshot batch with capture reason `near_close`
-- the nearest valid PREGAME odds snapshot before commence time.

- **`MarketCaptureReasons.NearClose`** now actually written (was reserved last slice).
- **`ClosingSnapshotCaptureService` / `IClosingSnapshotCaptureService`**: fetches current odds via the
  existing `OddsMarketClient`, gates on pregame, and persists a `near_close` batch through the existing
  ledger write path (`MarketSnapshotStore`). Reuses per-book h2h/spreads/totals normalization. Registered
  in DI. No HTTP endpoint (internal -- consistent with the prior market-memory services).
- **`ClosingCaptureResult`** + `ClosingCaptureStatuses`: written / deduped / skipped_started /
  skipped_no_market.
- **Dedupe key now includes the capture reason** (`MarketSnapshotNormalizer.ComputeDedupeKey`) so a
  `near_close` batch never collides with an `artifact_generation` batch carrying identical lines.

## Capture-reason semantics

- `artifact_generation` (unchanged): written during artifact generation, linked to the AgentRun.
- `near_close`: an explicit pregame capture intended to anchor the close. Never used for in-play odds;
  `latest` is never treated as close unless a batch is explicitly `near_close`.

## Pregame-only rules

The capture reads the odds event's commence time and:
- **pregame** (`now < commence`, market present) -> writes a `near_close` batch.
- **started** (`now >= commence`) -> `skipped_started`, nothing written (never contaminate close with
  in-play odds).
- **no market / off the board** (no odds event, e.g. postponed pulled from the board) ->
  `skipped_no_market`, nothing written (no stale close). Because commence is re-read from the provider on
  every capture, a rescheduled game anchors to its current start, never a stale one.

## Identity strategy

- preserves the **odds_api event id** (ProviderEventId) from the grounding -- the key to rejoin odds.
- preserves the **gamePk / external game id** when the caller provides it (ExternalGameId); null otherwise.
- does NOT require an AgentRun; optionally links one (LinkedAgentRunId) when provided.

## Idempotency / dedupe

Reuses the ledger dedupe key, now reason-scoped. Two `near_close` captures with identical lines (market
unchanged) -> deduped (one batch). A `near_close` whose lines moved -> a distinct batch. `closeObserved`
is a boolean (any `near_close` batch present), so multiple near_close batches do not break it; selecting
THE single close line is deferred to Artifact-vs-Close. Simpler approach chosen and documented.

## Movement / envelope interaction

- `MarketMovementDerivation.closeObserved` = false when no `near_close` batch exists; true when one does.
- `MarketMovementEnvelope.CloseObserved` mirrors derivation -- true only when derivation says so.
- `latest` is still never labeled close; representative movement remains deterministic and capped.
- No CLV / beat-close scoring.

## Tests

- `ClosingSnapshotCaptureServiceTests` (+6, full odds-parse path + in-memory DB): pregame writes a
  `near_close` batch with h2h/spreads/totals lines (odds event id + gamePk preserved, no AgentRun
  required); started game -> skipped_started, no write; empty market -> skipped_no_market, no write;
  duplicate near_close -> deduped; near_close does not collide with an identical artifact_generation
  batch (reason in the dedupe key); end-to-end: artifact_generation batch -> derivation closeObserved
  false -> near_close capture -> derivation + envelope closeObserved true.
- Existing ledger/query/derivation/envelope tests preserved (dedupe-key change is reason-consistent).
- Full `DevCore.Api.Tests` suite: **836 passed / 0 failed** (was 830; +6). No regressions.
- No Python/frontend change.

## Verification

- Behavior fully covered by deterministic tests, including the closeObserved false->true flip.
- Live DB cross-check: the ledger holds only `artifact_generation` batches and **zero** `near_close`
  today, so `closeObserved` is correctly false everywhere (no overclaim). Run e2b5433e / ExternalGameId
  824261 is past commence, so a near-close capture would `skipped_started`.
- **Live capture trigger skipped due to timing**: there is no endpoint to invoke the internal capture
  service this session, and the verified game has already started. The pregame write path is proven by
  the deterministic tests (which exercise the real odds parsing + ledger write + dedupe + closeObserved
  flip). No provider call occurs in tests (fake odds handler).

## What did not ship

No artifact JSON / buyer surface change; no movement surfaced; no CLV / artifact-vs-close scoring; no
scheduled capture / polling job; no provider historical integration; no endpoint; no prompt / confidence
/ lean / posture / reconciliation change.

## Risks / follow-ups

- Capture is manual/internal (no scheduler) -- a near_close batch only exists when something invokes the
  service. A future **Market Snapshot Scheduler v1** (or a thin dev endpoint/script) would capture
  near_close automatically before start.
- `closeObserved` is boolean; selecting the single canonical close line for CLV is **Artifact-vs-Close
  Calibration v1**.
- MarketMovement artifact DTO surfacing is still deferred.
- Provider quota/cost: near_close adds one paid odds call per capture (only when invoked).
- Untracked harness byproducts (8: calibration reports + artifact JSONs) remain in the vault tree,
  referenced by prior docs -- left outside this commit; disposition is the user's.

## Recommended next slice

**Market Snapshot Scheduler v1** -- a minimal trigger (thin dev endpoint or script) that invokes the
near-close capture for tracked pregame games, so multi-batch history (and real closeObserved) accrues
without manual calls. Alternatively **Artifact-vs-Close Calibration v1** (read-only), **MarketMovement
Artifact Projection v1**, or **Reconcile Directional-Contrast Cohort v1** once games 200013-200022 are
Final.
