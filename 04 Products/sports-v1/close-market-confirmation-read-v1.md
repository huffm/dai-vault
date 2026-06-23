# Close-Market Confirmation Read v1

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified (TDD). Code only (no schema/migration). Read-only. No
artifact/buyer/prompt/confidence/lean/posture/reconciliation change; no scheduler; not full CLV.
**implements:** the recommended next slice from `tracked-near-close-target-resolver-v1.md`. Turns the now-
joined artifact_generation + near_close batches into a buyer-first market-confirmation read.

## What shipped

A read-only comparison of the artifact-generation market state vs the near-close market state, relative to
the artifact's directional lean.

- **`CloseMarketConfirmationCalculator`** (pure): given lean side + generation/close consensus + lean-side
  implied probability, returns moved_toward_lean / moved_against_lean / neutral / not_comparable.
- **`CloseMarketConfirmationResult`** (read DTO): run + game identity, both batch ids + fetch times,
  consensus at generation/close + flip, lean-side implied probability at generation/close, the three
  movement booleans + direction, status, a buyer-safe summary, a diagnostic note, the CLV boundary, and an
  optional read-only `ArtifactEvalStatus`.
- **`CloseMarketConfirmationService`**: loads the run (lean), its artifact_generation batch, the joined
  near_close batch (by provider event id + gamePk when present), and an optional evaluation row; computes
  the verdict. Mutates nothing.
- **Dev-gated read endpoint** `GET /api/agent-runs/{agentRunId}/close-market-confirmation` (tenant-scoped,
  same identity path as the evaluation read). Internal -- not a buyer surface.

## Why this is not full CLV

The artifact carries a directional team lean (home/away), not a formal market selection (market type,
side, line, price). So true closing-line value is not evaluable: `TrueClvEvaluable` is always false and a
`ClvBoundaryNote` says so. This v1 compares the safest directional signal -- moneyline consensus +
lean-side implied probability. Spread/total comparison stays `not_comparable` unless the artifact ever
carries an explicit spread/total selection.

## Buyer-first language rules

Internal buyer-safe phrases (NOT surfaced to buyer UI yet):
- "Market moved toward this read." (moved_toward_lean)
- "Market moved against this read." (moved_against_lean)
- "Market stayed mostly neutral." (neutral)
- "No close-market comparison available yet." (no near_close / no lean / not_comparable / no generation)

## Comparison semantics

Primary: lean-side implied probability movement from generation to close. delta >= +1pp -> toward;
<= -1pp -> against; otherwise neutral. Fallback (no probabilities): consensus flip -- close consensus
becomes the lean side (and generation was not) -> toward; generation was the lean and close is not ->
against; otherwise neutral. Neither probabilities nor consensus -> not_comparable.

Examples: lean away + away implied prob rises -> moved_toward_lean; falls -> moved_against_lean; consensus
flips to the lean -> toward; flips off the lean -> against; no meaningful change -> neutral.

## Status model

`observed` (generation + near_close + lean all present and comparable); `no_generation_snapshot`;
`no_near_close_snapshot`; `no_artifact_lean`; `not_comparable`; `unavailable` (run not found).

## Identity join behavior

The near_close batch is joined to the run's artifact_generation batch by `Provider` + `ProviderEventId`,
and additionally by `ExternalGameId`/gamePk when the generation batch carries one (it does for tracked
captures). This relies on Tracked Near-Close Target Resolver v1 having preserved both identities.

## Optional outcome join behavior

If an `AgentRunEvaluation` exists for the run, its `EvalStatus` (correct/incorrect/inconclusive) is
surfaced read-only as `ArtifactEvalStatus`. The service reads only -- it does not change reconciliation,
outcome grading, or confidence. Verified by a test asserting the evaluation row count is unchanged.

## Tests

- `CloseMarketConfirmationTests`: pure calculator (probability rise->toward, fall->against, small->neutral,
  consensus flip toward/away, stable consensus->neutral, no signal->not_comparable) + service (observed
  with movement, probability fall->against, no_near_close, no_generation, no_artifact_lean, optional
  outcome join is read-only). 13 tests.
- Existing market-memory + reconciliation tests preserved.
- Full `DevCore.Api.Tests` suite: **861 passed / 0 failed** (was 848; +13). No regressions.
- No Python/frontend change.

## Live verification

`GET /api/agent-runs/e5b5433e/close-market-confirmation` (gamePk 824585): `status=observed`, `lean=away`,
`direction=neutral` -- the lean-side implied probability was 0.524 at both generation and near-close, so no
movement (honest neutral). Joined correctly: same ProviderEventId `22bf5f58...`, gamePk `824585`, both
batch ids present. `trueClvEvaluable=false`; summary "Market stayed mostly neutral." No
artifact/buyer/prompt/confidence/posture/reconciliation behavior changed.

## What did not change

No scheduler/polling/background job; no movement surfaced to artifact JSON; no buyer UI/copy; no prompt,
confidence, lean, posture, or reconciliation change; no outcome grading change. Read-only.

## Risks / follow-ups

- True CLV requires a concrete market selection (type/side/line/price) on the artifact -- a larger future
  change to the artifact contract.
- Outcome-calibrated read (market-moved-toward-lean vs settled outcome accuracy) after settlement.
- Market Snapshot Scheduler v1 to accrue near_close automatically.
- MarketMovement / confirmation artifact projection on the internal read DTO.
- Provider quota/cost: unchanged (read-only; no provider call).
- Vault hygiene: 10 untracked harness byproducts referenced by prior committed docs -- recommend a SEPARATE
  vault hygiene commit; kept out of this implementation commit.

## Decision Freshness Protocol seed

See `02 Platform/architecture/sources/decision-freshness-protocol-seed-v1.md` -- the future protocol that
generalizes this read into baseline-artifact -> refreshed-sources -> source-deltas -> freshness read.

## Recommended next slice

**Decision Freshness Architecture v1** (planning) or **Outcome-Calibrated Market Confirmation v1** once
games settle. Alternatively **Market Snapshot Scheduler v1** or **MarketMovement Artifact Projection v1**.
