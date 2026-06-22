# Market Snapshot Ledger v1

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code + one additive EF migration. Backward compatible. No
confidence/posture/lean logic, prompt-doctrine, buyer-UI, reconciliation, polling, or provider change.
**implements:** the recommended immediate next slice from `02 Platform/architecture/sources/
market-memory-layer-v1.md`. First persisted layer of DAI's market memory.

## What shipped

DAI now persists normalized odds market snapshots during artifact generation, reusing the multi-book
readings odds_api already returns (near-zero extra provider cost). This is observe/persist only -- line
movement, closing-line, and synthesis use are later slices.

- **Entities** (`DevCore.Domain/Agentic`): `MarketSnapshotBatch` (one capture of a game's market at one
  fetch time) + `MarketBookLine` (one book's one outcome -- the raw atom).
- **Migration** `20260622205819_AddMarketSnapshotLedger` -- two additive tables + indexes
  (unique DedupeKey; (Provider, ProviderEventId); LinkedAgentRunId; (Competition, FetchedAtUtc)). No
  change to existing tables.
- **`MarketSnapshotNormalizer`** (pure) -- expands per-book `MarketBookReading`s into normalized lines
  (h2h price, spread point, total line) and computes a deterministic dedupe key.
- **`IMarketSnapshotStore` / `MarketSnapshotStore`** -- writes a batch + lines, dedupes by DedupeKey.
- **Capture path:** `OddsMarketClient` now surfaces the raw per-book readings + per-book providerUpdatedAt;
  `SportsRetriever` builds a `MarketSnapshotDraft` (carrying the odds_api event id) onto
  `SportsRetrievalOutput`; `AgentRunService` passes it out on `AgentRunExecution`; the controller persists
  it after the run is saved, linked by `AgentRunId`.

## Schema / entities

**MarketSnapshotBatch:** key/id, TenantKey, Competition, Provider, ProviderEventId, ExternalGameId,
HomeTeamRef, AwayTeamRef, CommenceTimeUtc, FetchedAtUtc, LinkedAgentRunId, CaptureReason, DedupeKey,
BookCount, ConsensusSide, DisagreementRange, MedianHome/AwayImpliedProbability, MedianTotal,
MarketsPresent, CreatedAtUtc, Lines[].
**MarketBookLine:** key, MarketSnapshotBatchKey (fk, cascade), Bookmaker, MarketType (h2h/spreads/totals),
Side (home/away/over/under), Price, Point, ProviderUpdatedAt, CreatedAtUtc.

## Identity strategy

The batch carries BOTH identities, resolving the planning finding that the MLB path discarded the odds
event id:
- **ProviderEventId** = the odds_api event id -- captured here (previously discarded in `SportsRetriever`);
  it is what rejoins successive odds fetches for the same game.
- **ExternalGameId** = the statsapi gamePk -- the run's canonical identity for AgentRun + reconciliation,
  unchanged.
The odds event identity is captured for the ledger ONLY; it is NOT used as the run's reconciliation key.

## Snapshot semantics

- `FetchedAtUtc` = when DAI captured (server time); `providerUpdatedAt` (per line) = bookmaker last_update.
  They are distinct and both stored.
- This is the run-linked CURRENT snapshot. No first/latest/close designation and NO movement derivation
  this slice (deferred). CaptureReason = `artifact_generation`.

## Dedupe / idempotency

DedupeKey = SHA-256 over provider + provider event id + the sorted normalized lines
(marketType:side:bookmaker:price:point:providerUpdatedAt). It EXCLUDES FetchedAtUtc, so re-capturing
identical market state (e.g. an artifact-generation retry) dedupes (no duplicate batch), while any
price/point/providerUpdatedAt change yields a new batch. Enforced by a unique index + a pre-write
existence check in the store.

## Write timing / fail-soft

Persisted in the controller AFTER the run row is saved (so the AgentRun exists; the batch links by
AgentRunId). The write is wrapped in try/catch: a ledger failure is logged and swallowed -- it never
fails a completed artifact or changes the response. Absent market data -> no draft -> nothing written.

## What did not ship (deferred)

Line movement derivation; closing-line/near-close capture; scheduled polling / background jobs; provider
historical API; MarketMovementContext envelope; CLV/calibration; query API (Market Snapshot Query v1);
snapshotBatchId on the market_odds envelope (additive follow-up); non-MLB capture (the draft is built on
the MLB path; the entities are sport-agnostic).

## Tests

- `MarketSnapshotLedgerTests`: normalizer (line expansion h2h/spreads/totals, both identities + run link,
  consensus summary, dedupe-key stability + movement, empty readings) and store (write batch + lines,
  dedupe identical, keep distinct states) -- 7 tests.
- Full `DevCore.Api.Tests` suite: **800 passed / 0 failed** (was 793; +7). No regressions (MarketDepth,
  OddsMarketClient, retriever, controller integration all green).
- No Python/frontend change -> no py_compile / sports-app runs.

## Verification artifact (live)

Migration `20260622205819_AddMarketSnapshotLedger` applied to dev DB. Real MLB run `e2b5433e`: completed;
a `MarketSnapshotBatch` persisted with Provider=odds_api, **ProviderEventId=7eeb2a9f...** (odds event id),
**ExternalGameId=824261** (gamePk), LinkedAgentRunId=the run, BookCount=9, ConsensusSide=away,
MarketsPresent=h2h,spreads,totals, CaptureReason=artifact_generation; book lines h2h 18 / spreads 18 /
totals 9 (9 books). The market_odds SourceSignalEnvelope still present (observed); lean away @ 0.8,
posture monitor -- unchanged.

## Risks / follow-ups

- Spread/total prices are not captured (only the point), matching the existing Market Odds Depth readings;
  add if a later slice needs them.
- Provider quota/cost: run-triggered reuse is free; scheduled/near-close capture (later) is quota-gated.
- Provider can change a line without advancing providerUpdatedAt -> dedupe on the full tuple mitigates.
- Next slices: Market Snapshot Query v1; Market Movement Derivation v1; Closing Snapshot Capture v1;
  MarketMovementContext Envelope v1; CLV/calibration later.
- Untracked harness byproducts (this session's calibration reports + artifact JSONs) remain in the vault
  working tree and are referenced by prior docs -- left outside this commit; disposition is the user's.

## Recommended next slice

**Market Snapshot Query v1** -- a minimal internal read surface over the ledger (by game/run/provider/
time/marketType), the prerequisite for Market Movement Derivation v1. Alternatively **Reconcile
Directional-Contrast Cohort v1** once games 200013-200022 are Final.
