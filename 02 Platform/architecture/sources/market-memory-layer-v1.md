# Market Memory Layer v1 — Architecture and Requirements

**date:** 2026-06-22
**status:** FEATURE ARCHITECTURE + REQUIREMENTS PLAN. Vault-only. No application code, schema, migration,
runtime, prompt, buyer-UI, polling, persistence, provider, or background-job change. Implementation-ready,
transport-agnostic.
**builds on:** `market-odds-depth-v1.md`, `cross-sport-source-envelope-v1.md`,
`cross-sport-source-component-architecture-v1.md`, `grpc-inference-orchestration-boundary-v1.md`,
`calibration/reconciled-outcome-calibration-read-v1.md`.

## 1. Purpose

DAI reads the market well in the present (Market Odds Depth v1) but remembers nothing. A decision artifact
should eventually know not just what the market says now, but how it moved and whether the artifact agreed
with or beat later movement.

- **Current odds are a single snapshot** -- a point read at one fetch time.
- **Market memory is time-series evidence** -- the same game observed repeatedly, normalized and queryable.
- **Line movement and closing-line value require history** -- you cannot compute movement, consensus flip,
  or beat-the-close from one snapshot.
- **Artifact auditability requires knowing what the system saw at generation time** -- today the market
  context that grounded an artifact is not durably linked to the run as a queryable snapshot; only derived
  depth fields survive on the artifact.

This layer is DAI's owned, normalized, persisted market history -- the provider API stays the raw source.

## 2. Current Context

- **Market Odds Depth v1 (shipped):** `OddsMarketClient` fetches multi-book `h2h,spreads,totals` (current
  line only) from odds_api `/v4/sports/{key}/odds`; `MarketDepth` derives book count, consensus side,
  disagreement range, median implied probability, median total, and a lean-vs-market `AgreementWithLean`
  helper. Historical line movement was explicitly deferred (needs a paid snapshot endpoint).
- **Cross-Sport Source Envelope v1 (shipped):** `market_odds` is emitted as a `SourceSignalEnvelope`
  (status/depth/freshness/provenance). Freshness today is the bookmaker `UpdatedAt` (provider time);
  there is no structured DAI `FetchedAtUtc`, and the per-book lines are not persisted.
- **No persisted market history exists.** The multi-book readings are summarized into depth fields and the
  envelope, then discarded. There is no snapshot ledger, no movement, no close.
- **Reconciliation/calibration loop exists** ( `(mlb_statsapi, gamePk)` outcome key; Reconciled-Outcome
  Calibration Read v1 found confidence flat and accuracy tracking the home-lean base rate). Market movement
  is a natural future calibration dimension but has no data source yet.
- **Why now a full feature:** depth proved the market read is valuable; the next leverage is the time axis.
  This is large enough (schema + capture + movement + close + calibration) to deserve a feature plan, not
  another single envelope slice.

## 3. Feature Definition

DAI's **market memory layer**: observe, normalize, persist, and query odds over time, then derive movement
and comparison signals.

- **Market snapshot ledger** -- normalized per-book lines captured at known fetch times, keyed to game and
  (when generated during a run) to the AgentRun.
- **Market movement derivation** -- read-only deltas between snapshots (first/latest/close): moneyline,
  spread, total, implied-probability movement, consensus flips, convergence/divergence.
- **Artifact-vs-market comparison** -- did the artifact lean agree with the market at generation time?
- **Artifact-vs-close comparison** -- did the artifact lean beat the closing market (CLV, internal)?
- **Future synthesis augmentation** -- a `MarketMovementContext` envelope feeding the analyzer/synthesis
  once evidence supports it.
- **Future calibration dimension** -- group reconciled outcomes by movement-toward/against-lean and
  beat-close, after outcomes settle.

## 4. Non-Goals

Explicitly deferred: implementation; schema/migrations; polling jobs; dashboards; buyer-facing CLV/claims;
confidence tuning; automatic decision/lean changes from movement; paid historical-odds provider
integration; gRPC; a full odds warehouse; covering every sport/book/market exhaustively; in-play/live
trading data.

## 5. Core Use Cases

1. Capture market state when an artifact is generated (run-linked snapshot batch).
2. Compare a later run against earlier snapshots for the same game.
3. Derive line movement between snapshots (first -> latest).
4. Identify consensus flips and market convergence/divergence over time.
5. Capture near-close / closing market state for a tracked game.
6. Compare artifact lean to the market at generation time.
7. Compare artifact lean to the closing market (internal beat-close).
8. Support calibration after outcome reconciliation (movement as a slicing dimension).
9. Eventually provide `MarketMovementContext` to synthesis (grounded, read-only).

## 6. Domain Model (conceptual)

- **MarketEventIdentity** -- the join anchor for a game's market. Fields: competition/sport, internal
  `GameIdentity` facets (sourceProvider, externalGameId, scheduledStartUtc, season, homeTeamRef,
  awayTeamRef), the **odds_api event id** (critical -- see §11), and the sport-canonical id (MLB gamePk).
  Relationship: one per real game; joins snapshots to AgentRun and reconciliation. *Persisted now.*
- **MarketSnapshotBatch** -- one capture pass for one event at one fetch time. Fields: batchId,
  marketEventIdentity ref, fetchedAtUtc, captureMode (run/scheduled/near-close/manual/backfill), optional
  linked AgentRunId, provider, dedupeKey. *Persisted now.*
- **MarketBookLine** -- one book's one outcome in a batch (the raw normalized atom). Fields: batchId,
  bookmaker, marketType (h2h/spreads/totals), side (home/away/over/under), price, point, providerUpdatedAt.
  *Persisted now.*
- **MarketConsensusSnapshot** -- the `MarketDepth`-style summary of a batch (book count, consensus side,
  disagreement range, median implied prob, median total). *Derivable from book lines; may be cached.*
- **MarketMovementSummary** -- deltas across batches for an event (first/latest/close). *Derived later,
  read-only; not persisted v1.*
- **ArtifactMarketSnapshotLink** -- links an AgentRun/artifact to the batch captured at its generation.
  *Persisted now (or as a field on the batch).*
- **ClosingMarketSnapshot** -- the batch designated as "close" for an event (nearest valid pregame batch
  before start). *Derived/marked later.*
- **MarketMemoryQuery** -- read API over the ledger (by game/run/provider/time/marketType). *Implemented in
  the query slice; not an entity.*

## 7. Minimum Persisted Fields (viable v1 ledger)

Per snapshot batch + book line:

- competition / sport
- internal event identity (GameIdentity facets)
- provider (e.g. `odds_api`)
- provider event id (the odds_api event id)
- external game identity if available (MLB gamePk; sport-canonical id otherwise)
- homeTeamRef / awayTeamRef (normalized)
- commenceTime (provider scheduled start)
- fetchedAtUtc (DAI capture time)
- providerUpdatedAt (bookmaker last_update)
- bookmaker
- marketType: h2h / spreads / totals
- side: home / away / over / under
- price (American)
- point (spread handicap / total line; null for h2h)
- snapshotBatchId
- dedupeKey / sourceHash (see §10)
- linkedAgentRunId (when captured during generation; null otherwise)

## 8. Derived Fields and Metrics (derived, not necessarily persisted)

first observed line; latest line; near-close line; closing line; moneyline movement; spread movement;
total movement; consensus side; consensus flip (boolean + when); book count; disagreement range; market
convergence/divergence (disagreement narrowing/widening); implied-probability movement; market moved
toward artifact lean; market moved against artifact lean; artifact beat close; artifact lost to close;
stable vs unstable market (movement magnitude band). All read-only; computed from the ledger; none mutate
a decision in v1.

## 9. Snapshot Capture Strategy

Capture modes:

- **run-triggered** -- on artifact generation, persist the multi-book readings already fetched by
  `OddsMarketClient` (near-zero marginal provider cost -- the call already happens). *v1 core.*
- **scheduled pregame** -- coarse snapshots for tracked games (e.g. a few times between listing and start).
  *Later slice; quota-gated.*
- **near-close** -- one capture close to start time to anchor the closing line. *Later slice.*
- **manual/dev** -- a script to capture on demand for a game. *Cheap to add alongside v1.*
- **backfill** -- optional, from a provider historical endpoint, as validation only. *Deferred.*

Conservative v1 cadence: **capture on artifact generation only**, plus an optional manual capture. No
scheduled polling, no minute-by-minute. Discussion points: quota/cost (every extra fetch is paid odds_api
quota -- run-triggered reuses the existing call, so it is free); idempotency + duplicate avoidance via the
dedupeKey (§10); stale provider timestamps (providerUpdatedAt unchanged across fetches is normal -- store
fetchedAtUtc regardless); postponement/cancellation (mark the event, do not treat a stale line as close).

## 10. Opening / Current / Latest / Closing Semantics

- **first observed** is NOT the true opening line unless the provider explicitly says so -- label it
  `first_observed`, never `open`.
- **current** = the snapshot used at a given artifact's generation (the run-linked batch).
- **latest** = the most recent DAI snapshot for the event.
- **close** = the nearest valid PREGAME snapshot before scheduled start; chosen by capture time vs
  commenceTime, never an in-play line.
- **live/in-play** odds must be excluded or separately flagged (a batch captured after commenceTime for a
  started game is not pregame market memory).
- **postponed/rescheduled** games: the prior "close" is invalidated if start time moves; re-anchor on the
  new start. Never grade beat-close against a stale, pre-postponement line.

## 11. Identity and Join Strategy

Joins: provider event id <-> internal `GameIdentity` <-> MLB gamePk <-> AgentRun <-> artifact <-> outcome
reconciliation <-> calibration reads.

**Critical finding (grounded in code):** today `SportsRetriever` **discards the odds_api event id for MLB**
and keeps only the statsapi gamePk as the canonical identity (Outcome Reconciliation Contract v1). That is
correct for reconciliation, but market memory MUST additionally persist the **odds_api event id**, because
re-fetching/joining future odds for the same game keys on the provider's own event id, not the gamePk. So
`MarketEventIdentity` carries BOTH: gamePk (joins to AgentRun + reconciliation) and the odds event id
(joins successive odds fetches). The Market Snapshot Ledger v1 slice must stop discarding the odds event
id on the MLB path (a small, additive capture, not a behavior change to lean/confidence).

Risks: provider event id mismatch or re-issue; team-name normalization drift; doubleheaders (disambiguate
by commenceTime); postponed games (start-time change breaks close anchoring); neutral sites; changing start
times. Rule: never fabricate a join; an unmatched snapshot is stored with whatever identity it has and
marked needs-review, never force-joined.

## 12. Interaction with SourceSignalEnvelope

- The current `market_odds` envelope represents the **current/depth** snapshot (one fetch).
- A future `market_movement` envelope (group `market_movement`, already reserved in `SourceSignalTaxonomy`)
  represents derived **historical** context from the ledger.
- Source envelopes should carry the **snapshotBatchId** so an artifact's market read is traceable to the
  exact persisted batch.
- Market memory feeds SourceDepth / EvidenceSufficiency only AFTER explicit rules exist; movement is not a
  new grounded breadth signal by default. No confidence tuning is recommended here.

## 13. Interaction with Synthesis (staged)

- **Stage 1: persist only** -- the ledger writes; nothing reads it into a decision.
- **Stage 2: derive movement read-only** -- movement summaries computed and inspectable; not in the prompt.
- **Stage 3: expose `MarketMovementContext`** to the analyzer/synthesis as factual data (no doctrine
  change), only when grounded in real snapshots.
- **Stage 4: calibration + confidence** -- use movement/beat-close only after reconciled-outcome evidence
  shows it discriminates.

Clarifications: movement is **evidence, not proof**; synthesis may mention movement only when grounded in
persisted snapshots; no buyer-facing CLV/claims until validated.

## 14. Storage and Retention

Expected growth: book lines dominate (books x markets x sides x batches). Run-triggered-only v1 keeps this
modest (one batch per run). Retain raw book lines (the audit atom) and derive summaries on read; optionally
cache consensus per batch. Pruning/archiving: defer until volume warrants; design the dedupeKey + indexes
now so pruning is possible later. Indexes: (eventIdentity, fetchedAtUtc), (linkedAgentRunId),
(provider, providerEventId). Dedupe: a deterministic key over (provider, providerEventId, bookmaker,
marketType, side, providerUpdatedAt, point, price) so re-fetching the same unchanged line does not create a
new row within a batch. Tenant boundaries: market data is tenant-agnostic reference data; keep it shared,
but design so a tenant scope can be added later. Avoid warehouse bloat: no per-second capture, no every
book forever -- capture what a run used plus deliberate scheduled/close snapshots later.

## 15. Observability and Operations

Capture success/failure logs per event; provider quota usage (calls made, remaining); snapshot/batch
counts; duplicate-suppressed counts; last successful capture by competition; stale-provider-data warnings
(providerUpdatedAt not advancing); capture latency; cost tracking (paid odds_api calls vs free
run-reuse); manual repair/backfill notes. All read-only telemetry; no dashboard.

## 16. Risks and Edge Cases

Provider quota/cost (mitigate: run-reuse first, coarse scheduling later); provider changes a line without
advancing UpdatedAt (store fetchedAtUtc regardless; dedupe on full tuple); missing/partial books; partial
markets (h2h present, totals absent); in-play odds contamination (exclude post-commence); event identity
mismatch (needs-review, never force-join); line-movement overinterpretation and false confidence from
movement (stage gating; movement is evidence not proof); storing too much too early (run-triggered v1
only); treating first observed as the true opening line (label `first_observed`).

## 17. Recommended Implementation Slices (ordered)

1. **Market Snapshot Ledger v1** -- normalized snapshot schema + write the multi-book readings already
   fetched during artifact generation; preserve the odds_api event id on the MLB path. *Goal:* durable
   run-linked market history. *Non-goals:* movement, close, polling, synthesis, confidence. *Files:*
   new ledger entities + migration, `OddsMarketClient`/`SportsRetriever` capture hook, persistence.
   *Tests:* normalized write, dedupe, run link, fail-soft. *Verify:* one live MLB run writes a batch +
   book lines joined to the AgentRun.
2. **Market Snapshot Query v1** -- read API by game/run/provider/time/marketType. *Tests:* query shapes;
   *Verify:* fetch the batch written in slice 1.
3. **Market Movement Derivation v1** -- read-only first/latest/close deltas + consensus flip. *Verify:*
   deterministic movement over seeded snapshots.
4. **Closing Snapshot Capture v1** -- near-start capture + close semantics + postponement handling.
5. **MarketMovementContext Envelope v1** -- emit a `market_movement` `SourceSignalEnvelope` from history,
   carrying snapshotBatchId; observed-only.
6. **Artifact-vs-Close Calibration v1** -- read-only calibration dimension (beat-close, moved-toward/
   against-lean) over reconciled outcomes.
7. **Synthesis Market Movement Integration v1** -- expose movement to the analyzer; only after slice 6
   shows it discriminates.

## 18. Recommended Immediate Next Slice

**Market Snapshot Ledger v1.** It comes first because every later slice (query, movement, close,
calibration, synthesis) needs persisted snapshots to exist, and it can be built at near-zero provider cost
by persisting the multi-book readings the run already fetches. It must NOT include movement derivation,
closing semantics, scheduled polling, synthesis exposure, buyer surfaces, or any confidence/lean change.

Acceptance criteria: a normalized snapshot batch + per-book lines are written on artifact generation for
MLB; the batch carries both the statsapi gamePk and the odds_api event id; the batch links to the
AgentRunId; re-running the same fetch does not duplicate unchanged lines (dedupeKey); writing is fail-soft
(a ledger failure never fails the run or changes the artifact); `dotnet test` green; one live MLB run shows
a persisted batch joined to its run.

Verification strategy: TDD on the normalization + dedupe (pure), plus an integration test that a run writes
the expected rows; then a live MLB run + a read-only DB query confirming the batch, book lines, gamePk +
odds event id, and AgentRun link. Confirm no change to lean/confidence/posture/buyer/reconciliation.

## 19. Ready-to-Paste Execution Prompt

```
Slice: Market Snapshot Ledger v1

Goal: persist a normalized market snapshot ledger written during artifact generation, joined to the
AgentRun and to game identity. Observe/persist only -- no movement, no close, no polling, no synthesis,
no confidence/lean change. Reuse the multi-book odds the run already fetches (near-zero extra provider
cost). Preserve the odds_api event id on the MLB path (currently discarded).

Skills gate: dai-skill-router, dai-grill-with-vault, superpowers:test-driven-development,
dai-test-discipline, systematic-debugging, verification-before-completion, dai-agent-handoff.

Repos: dai (code), dai-vault (docs).

Preflight: git status/log both repos; confirm clean/synced; confirm services (devcore-sql, DevCore.Api,
agent-service) state; do not push.

Inspect first:
- dai-vault/02 Platform/architecture/sources/market-memory-layer-v1.md (this plan -- sections 6,7,10,11,14)
- dai/platform/dotnet/DevCore.Api/Sports/OddsMarketClient.cs (multi-book parse, MarketBookReading)
- dai/platform/dotnet/DevCore.Api/AgentRuns/MarketDepth.cs
- dai/platform/dotnet/DevCore.Api/AgentRuns/SportsRetriever.cs (mlb block; odds event id currently discarded)
- dai/platform/dotnet/DevCore.Api/Sports/GameIdentity.cs (GameIdentityContext, gamePk)
- dai/platform/dotnet/DevCore.Api/AgentRuns/SportsRetrievalOutput.cs
- existing EF context / migration pattern (AppDbContext, prior AgentRun migrations)

Implementation scope:
- Entities: MarketSnapshotBatch (batchId, eventIdentity facets incl. odds_api event id + gamePk,
  provider, fetchedAtUtc, captureMode, linkedAgentRunId?, dedupeKey) + MarketBookLine (batchId,
  bookmaker, marketType h2h/spreads/totals, side, price, point?, providerUpdatedAt).
- Expose the per-book MarketBookReading from OddsMarketClient up to the capture point (it is currently
  summarized then dropped) WITHOUT changing MarketDepth/depth/envelope behavior.
- Stop discarding the odds_api event id on the MLB market path; carry it for the ledger only (do NOT
  change the reconciliation identity, which stays gamePk).
- Write the batch + lines during artifact generation, fail-soft (a ledger write failure is logged and
  swallowed; the run and artifact are unaffected).
- Deterministic dedupeKey over (provider, providerEventId, bookmaker, marketType, side, point, price,
  providerUpdatedAt) so unchanged lines do not duplicate within a batch.

Schema/migration guidance: one additive EF migration adding the two tables + indexes
((eventIdentity, fetchedAtUtc), (linkedAgentRunId), (provider, providerEventId)). SQL Server (devcore).
No change to existing tables. Additive only; backward compatible.

Idempotency: re-fetching the same unchanged lines within a run must not create duplicate rows (dedupeKey).

Tests (TDD): normalization of MarketBookReading -> MarketBookLine; dedupeKey stability; batch links to
AgentRunId; both gamePk and odds event id captured; fail-soft when persistence throws. Run dotnet test
(DevCore.Api.Tests) to green. No Python/frontend change expected.

Verification: one live MLB run; read-only DB query showing the batch + book lines, gamePk + odds event
id, AgentRun link; confirm dedupe on a second fetch; confirm lean/confidence/posture/buyer/reconciliation
unchanged.

Docs: dai-vault/04 Products/sports-v1/market-snapshot-ledger-v1.md (schema, capture hook, dedupe,
fail-soft, verification, deferred). Append a compact current-slice.md handoff entry.

Commit: dai and dai-vault SEPARATELY (conventional commits). Do not push unless asked; state hashes.
Handoff: result / repo state / files changed / tests / verification / commit hashes / next slice
(Market Snapshot Query v1). Deliver the final handoff in ONE fenced code block.
```
