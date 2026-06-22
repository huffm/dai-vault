# Market Snapshot Query v1

**date:** 2026-06-22
**status:** IMPLEMENTED + verified (TDD). Code only (no schema/migration). Read-only. No write-path,
artifact, prompt, buyer-UI, confidence/lean/posture, or reconciliation change. No provider calls.
**implements:** the recommended next slice from `market-snapshot-ledger-v1.md`. Prerequisite for Market
Movement Derivation v1.

## What shipped

A minimal, internal, read-only query surface over the persisted market snapshot ledger.

- **`MarketSnapshotQuery`** (filter record): composable, all-optional filters -- LinkedAgentRunId,
  ExternalGameId, ProviderEventId, Provider, MarketType, FetchedFrom, FetchedTo.
- **`MarketSnapshotBatchView` / `MarketBookLineView`** (read DTOs): explicit, stable shapes carrying the
  batch summary (provider, both identities, fetchedAtUtc, consensus summary, marketsPresent) and the
  per-line detail needed by movement derivation later.
- **`IMarketSnapshotQueryService` / `MarketSnapshotQueryService`**: a small query service over
  `AppDbContext` (read-only, `AsNoTracking`). Registered in DI. No controller/route added (internal/dev
  surface; no existing route pattern required one).

## Query semantics

- Filters are composable; null = do not filter on that dimension.
- `MarketType`, when set, restricts the returned lines to that type AND excludes batches that carry no
  line of that type.
- Deterministic ordering: batches by `FetchedAtUtc` then surrogate key; lines by MarketType, Side,
  Bookmaker, Point, Price, ProviderUpdatedAt (ordinal/numeric).
- Empty result is a normal success (an empty list), never an error.

## Boundaries (not changed / not done)

- No movement derivation, no first/latest/close computation (Market Movement Derivation v1 later).
- No provider calls, no polling, no scheduled jobs.
- No change to the ledger write path, artifact generation, prompts, buyer surface, lean/confidence/
  posture, or reconciliation.
- No HTTP endpoint (kept internal; add later only if a consumer needs it).

## Tests

- `MarketSnapshotQueryServiceTests` (+7, in-memory `AppDbContext`): by linked AgentRunId; by
  ExternalGameId; by ProviderEventId; marketType filtering (restricts lines + excludes batches lacking
  the market); fetched time-range; deterministic batch + line ordering; empty result is success.
- Existing `MarketSnapshotLedgerTests` preserved.
- Full `DevCore.Api.Tests` suite: **807 passed / 0 failed** (was 800; +7). No regressions.
- No Python/frontend change.

## Verification

- Service behavior fully covered by deterministic seeded tests.
- Live DB cross-check (run `e2b5433e` from Ledger v1): the persisted batch is reachable by
  LinkedAgentRunId, ExternalGameId=824261, and ProviderEventId=7eeb2a9f...; h2h lines return in the
  service's deterministic order (MarketType, Side, Bookmaker). The query reads only the DB -- no provider
  call occurs.

## Recommended next slice

**Market Movement Derivation v1** -- read-only first/latest/close deltas + consensus flip over batches
returned by this query (group by ExternalGameId, ordered by FetchedAtUtc). Alternatively **Reconcile
Directional-Contrast Cohort v1** once games 200013-200022 are Final.
