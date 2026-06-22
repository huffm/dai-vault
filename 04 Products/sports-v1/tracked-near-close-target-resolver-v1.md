# Tracked Near-Close Target Resolver v1

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code only (no schema/migration). Read + capture by provider
event id. No artifact/buyer/prompt/confidence/posture/reconciliation change; no scheduler; no CLV.
**implements:** the recommended next slice from `near-close-capture-trigger-v1.md`. Fixes the previous
slice's gap where near_close batches had null gamePk (team/date targets), so artifact-vs-close was not
joinable.

## Why this slice exists

Near-Close Capture Trigger v1 made capture operational but the live demo used team/date targets, producing
near_close batches with the odds event id but null gamePk. Before a scheduler, DAI needs a tracked target
resolver that finds games it already observed during artifact generation and captures near_close while
preserving full identity (provider event id + gamePk + AgentRun) -- so near_close joins back to the same
game. Core principle: do not automate capture until target identity is reliable.

## Target resolver behavior

`TrackedNearCloseTargetResolver` (read-only, no provider calls) finds eligible targets from the market
ledger:

- this tenant's `artifact_generation` batches, provider `odds_api`, with a non-null `ProviderEventId`,
  and `CommenceTimeUtc` in the future (pregame).
- one target per provider event id (the latest artifact_generation batch for that event).
- each target carries: competition, provider, providerEventId, externalGameId/gamePk, agentRunId,
  home/away refs, commenceTimeUtc, the eastern game date (for the provider bracket), the source
  artifact_generation batch id, and an `AlreadyHasNearClose` flag.
- batches without a provider event id are NOT eligible (identity not reliable enough to capture).

## Identity strategy

A near_close capture produced from a tracked target preserves:

- **ProviderEventId** (odds_api event id) -- guaranteed identical because capture matches the odds event by
  id, not team name.
- **ExternalGameId / gamePk** -- carried from the source artifact_generation batch.
- **AgentRunId** -- carried when the source run had one.
- competition, commenceTimeUtc, home/away refs, capture reason `near_close`.

Acceptance (verified live): a generated artifact's `artifact_generation` batch and its `near_close` batch
are joinable by provider event id AND gamePk AND AgentRun.

## Provider event id vs team/date matching

`OddsMarketClient` gained `GetBaseballSpreadByEventIdAsync` -- it fetches the same odds endpoint and
selects the event by `id == providerEventId` (no team-name matching). `ClosingSnapshotCaptureService`
gained `CaptureNearCloseByEventIdAsync` (shares the same gate + persist core as the team-name path). The
tracked trigger uses the event-id path, so it never relies on ledger team-ref slugs to drive provider
team-name matching. The explicit team/date trigger from the prior slice is unchanged (fallback).

## Endpoint / trigger shape

- New dev-gated endpoint `POST /api/agent-runs/near-close-capture/tracked` with `{ force }`, tenant-scoped
  via the same identity path as reconcile.
- `NearCloseCaptureTrigger.CaptureTrackedAsync` resolves eligible tracked targets and captures each by
  event id. Result summary: `eligible`, `attempted`, `written`, `deduped`, `skippedStarted`,
  `skippedNoMarket`, `skippedExists`, `skippedMissingProviderEventId` (0 -- resolver pre-filters), plus
  per-game results.

## Idempotency

- resolver flags `AlreadyHasNearClose`; the tracked trigger skips those (`skipped_exists`) unless `force`.
- with `force`, the capture service's reason-scoped dedupe key decides (identical lines dedupe).
- no duplicate explosion.

## Tests

- `TrackedNearCloseResolverTests` (+7): resolver finds a pregame artifact_generation target; skips missing
  provider event id; carries gamePk + AgentRun; flags AlreadyHasNearClose; excludes past commence. Tracked
  trigger: captures by event id preserving gamePk + provider event id + run; skips already-captured
  without force.
- `NearCloseCaptureTriggerTests` updated for the new ctor (resolver injected); existing assertions
  preserved.
- Full `DevCore.Api.Tests` suite: **848 passed / 0 failed** (was 841; +7). No regressions.
- No Python/frontend change.

## Live verification

Generated one fresh MLB artifact (run e5b5433e) for an upcoming pregame game -> an `artifact_generation`
batch persisted with ProviderEventId `22bf5f58...`, ExternalGameId `824585` (gamePk), 45 book lines.
`POST /api/agent-runs/near-close-capture/tracked` -> `eligible=1, written=1`. The resulting `near_close`
batch carries the SAME ProviderEventId `22bf5f58...`, ExternalGameId `824585`, LinkedAgentRunId, and 45
lines. `closeObserved` is now TRUE for gamePk 824585 (a near_close batch joins it by gamePk) -- the gap
from the prior slice is fixed. No team-name matching occurred; no artifact/buyer/prompt/confidence/posture/
reconciliation behavior changed.

## What did not ship

No scheduler / recurring polling / background job; no movement surfaced to the artifact JSON; no buyer
change; no CLV / artifact-vs-close scoring; no provider historical integration.

## Follow-ups

- **Market Snapshot Scheduler v1** -- call this tracked trigger automatically for pregame games near start.
- **Artifact-vs-Close Calibration v1** -- now unblocked: a game can have both an artifact_generation and a
  gamePk-joined near_close batch (CLV / beat-close).
- **MarketMovement Artifact Projection v1** -- surface the movement/close envelope on the internal artifact
  read DTO.
- Provider quota/cost: each captured tracked target is one paid odds call.
- Vault hygiene: 8 untracked harness byproducts (calibration reports + artifact JSONs) referenced by prior
  committed docs -- recommend a SEPARATE vault hygiene commit; kept out of this implementation commit.

## Recommended next slice

**Artifact-vs-Close Calibration v1** -- now that artifact_generation and near_close batches join by gamePk,
compute the read-only beat-close / moved-toward-lean dimension over reconciled outcomes. Alternatively
**Market Snapshot Scheduler v1**, **MarketMovement Artifact Projection v1**, or **Reconcile
Directional-Contrast Cohort v1** once games 200013-200022 are Final.
