# Near-Close Capture Trigger v1

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code only (no schema/migration). Makes near-close capture
operational. No scheduler/polling/background job; no artifact/buyer/prompt/confidence/posture/
reconciliation change; no CLV.
**implements:** the recommended next slice from `closing-snapshot-capture-v1.md`. Closing Snapshot Capture
v1 was inert (no caller); this makes it invocable.

## Trigger shape

- **`NearCloseCaptureTrigger` / `INearCloseCaptureTrigger`** (orchestration service): iterates explicit
  pregame targets, invokes `IClosingSnapshotCaptureService` per target, applies a pre-flight "already has
  near_close" skip, and aggregates counts. Not a scheduler -- it captures the supplied targets once.
- **Dev-gated endpoint** `POST /api/agent-runs/near-close-capture` on AgentRunsController, tenant-scoped
  via the same `identity.ResolveAsync` path as the reconcile endpoint (dev bypass-auth in development).
  Request: `{ games: [{ competition, homeTeam, awayTeam, gameDate, externalGameId?, agentRunId? }], force }`.

## Selection rules

- The caller supplies explicit game targets (the smallest fit; ledger auto-selection cannot reconstruct
  the provider's team-name inputs from stored team-ref slugs, so explicit targets are used).
- Idempotency pre-flight: if `externalGameId` is provided, `force` is false, and a `near_close` batch
  already exists for that game, the target is skipped (`skipped_exists`) without an odds call.
- All other gating (pregame-only, market presence) is delegated to `ClosingSnapshotCaptureService`.
- To tie a near_close batch to a game's gamePk-keyed movement, pass `externalGameId` (the gamePk). Without
  it, the batch still carries the odds_api event id but no gamePk.

## Input / output

Input: a list of game targets + `force`. Output `NearCloseTriggerResult`: `attempted`, `written`,
`deduped`, `skippedStarted`, `skippedNoMarket`, `skippedExists`, and a per-game list
(`{competition, homeTeam, awayTeam, externalGameId, status, batchId}`).

## Close semantics (delegated, unchanged)

- pregame (`now < commence`, market present) -> writes `near_close`.
- started (`now >= commence`) -> `skipped_started`, no write (never in-play).
- no market / off the board -> `skipped_no_market`, no stale close.
- `latest` is never treated as close; `closeObserved` counts only `CaptureReason=near_close`.

## Idempotency

Two layers: (1) the trigger's exists pre-flight (when `externalGameId` provided + not force) skips an
already-captured game without an odds call; (2) the capture service's dedupe key (reason + provider +
event id + sorted lines) dedupes identical re-captures. Verified live: re-running the same targets
returned `written=0, deduped=3` -- no duplicate explosion.

## Tests

- `NearCloseCaptureTriggerTests` (+5, full odds-parse path + in-memory): eligible pregame captured
  (`written=1`); started game skipped; no market -> `skipped_no_market`; existing near_close +
  not-force -> `skipped_exists` (no odds call); summary counts aggregate in one call
  (`attempted=2, written=1, deduped=1`).
- Existing closing-capture/ledger/query/derivation/envelope tests preserved.
- Full `DevCore.Api.Tests` suite: **841 passed / 0 failed** (was 836; +5). No regressions.
- No Python/frontend change.

## Live verification

- `POST /api/agent-runs/near-close-capture` for three upcoming MLB games -> `written=3`; the ledger gained
  three `near_close` batches (provider odds_api, distinct event ids, per-book lines h2h/spreads/totals).
- Re-running the same targets -> `written=0, deduped=3` (idempotent).
- closeObserved is now real: `near_close` batches exist in the ledger. The closeObserved false->true flip
  for a specific gamePk-keyed derivation is covered by deterministic tests (Closing Snapshot Capture v1);
  the only ledger game with a gamePk-tagged artifact_generation batch (824261) is past commence, so its
  live flip is timing-skipped. The live demo used team/date targets (no gamePk), so its near_close batches
  carry the odds event id but null gamePk -- pass `externalGameId` to tie them to a game's movement.
- No artifact JSON / buyer surface / prompt / confidence / posture / reconciliation behavior changed.

## What did not ship

No production scheduler; no recurring polling/background job; no movement surfaced to the artifact JSON;
no buyer change; no CLV / artifact-vs-close scoring; no provider historical integration.

## Follow-ups

- A scheduler (Market Snapshot Scheduler v1) could call this trigger automatically for tracked pregame
  games, so multi-batch history accrues without manual calls.
- MarketMovement artifact DTO projection still deferred.
- Artifact-vs-Close Calibration v1 (CLV) deferred -- needs both an artifact_generation and a gamePk-tagged
  near_close batch for the same game.
- Provider quota/cost: each captured target is one paid odds call.
- Vault hygiene: 8 untracked harness byproducts (calibration reports + artifact JSONs) are referenced by
  prior committed docs; recommend a SEPARATE vault hygiene commit to add them (or update the refs) -- they
  were deliberately kept out of this implementation commit.

## Recommended next slice

**Market Snapshot Scheduler v1** -- a minimal recurring/dev trigger that invokes this near-close capture
for tracked pregame games so closeObserved accrues automatically. Alternatively **Artifact-vs-Close
Calibration v1**, **MarketMovement Artifact Projection v1**, or **Reconcile Directional-Contrast Cohort v1**
once games 200013-200022 are Final.
