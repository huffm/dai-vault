---
title: "Outcome Reconciliation Follow-up v5"
type: "reconciliation"
date: "2026-07-01"
status: "no-op"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v5"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - settlement
  - statsapi
  - no-op
  - time-gated
related:
  - "06 Execution/outcome-reconciliation-follow-up-v4.md"
  - "04 Products/sports-v1/calibration/calibration-delta-v1.md"
---

# Outcome Reconciliation Follow-up v5 -- Evidence Report (no-op)

## status

Complete (no-op). The 10 remaining backlog games (824818 on 07-01 + nine 07-02 targeted) are all still
`Scheduled` / `Preview` -- **0 Final; 0 reconciled; 10 deferred**. First pitch for 824818 has not occurred;
the 07-02 slate is future. Non-paid; settlement-gated; no fabricated outcomes; no writes.

## start state (verified)

- `dai`: clean, synced, `1a2ddf2` (0/0). `dai-vault`: clean, synced, `34e79e3` (0/0) -- the v4 docs commit is
  present and pushed.
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4 regimes). No code/prompt/schema change this slice.
- Platform API up on `:5007` (`/health` -> ok) and `devcore-sql` up on `:1433` (left running from v4). FastAPI
  agent-service NOT started (reconciliation is a platform-only path, no model call).

## settlement probe (free StatsAPI, 10 remaining backlog gamePks)

`statsapi.mlb.com/api/v1/schedule?sportId=1&gamePks=...&hydrate=linescore` (free public schedule endpoint --
not a paid model call, no analysis run). Result: **0 Final; 10 non-Final.**

| gamePk | game (away @ home) | date | status | eligible? |
|---|---|---|---|---|
| 824818 | White Sox @ Orioles | 2026-07-01 | Scheduled / Preview | NO (first pitch not reached) |
| 824335 | Marlins @ Rockies | 2026-07-02 | Scheduled / Preview | NO (future) |
| 824416 | White Sox @ Guardians | 2026-07-02 | Scheduled / Preview | NO (future) |
| 824906 | Cardinals @ Braves | 2026-07-02 | Scheduled / Preview | NO (future) |
| 824093 | Rays @ Royals | 2026-07-02 | Scheduled / Preview | NO (future) |
| 822884 | Tigers @ Rangers | 2026-07-02 | Scheduled / Preview | NO (future) |
| 823935 | Padres @ Dodgers | 2026-07-02 | Scheduled / Preview | NO (future) |
| 823442 | Pirates @ Phillies | 2026-07-02 | Scheduled / Preview | NO (future) |
| 823765 | Reds @ Brewers | 2026-07-02 | Scheduled / Preview | NO (future) |
| 823119 | Angels @ Mariners | 2026-07-02 | Scheduled / Preview | NO (future) |

No scores posted on any game.

## partition

- **Final / eligible: 0.**
- **Non-Final / deferred: 10** (all `Scheduled` / `Preview`).
- **Ambiguous identity/status: 0** (all 10 resolved cleanly to a known gamePk with a clear non-Final status).

## reconciliation actions

**None.** 0 of 10 Final -> 0 eligible. No `/reconcile` or per-run `/outcome` calls; no fabricated outcomes; no
non-final reconcile. Per the hard constraint, outcomes were not inferred from any score because no game is Final.

## 824818 collision status (pre-checked, not acted on)

Re-confirmed via `/rows` (active runs only): gamePk **824818 has 2 active runs** -- non-backlog `3ade423e`
and backlog soak `28bd433e`, both lean-null. When 824818 reaches `Final`, identity `/reconcile` would return
`MultipleMatches` and write nothing; reconcile **only** `28bd433e` via per-run `/{28bd433e}/outcome`
(same doctrine as 825066 in v4). Not acted on this slice (not Final).

## DB / API verification (before == after, nothing settled)

Read-only `/prompt-route-calibration/metrics` and `/rows` before and after the probe -- identical (no write path
was exercised):

| metric | value (unchanged) |
|---|---|
| totalRows | 263 |
| reconciledRows | 84 |
| unreconciledRows | 169 |
| noDecisionRows | 10 |
| matchedRows | 51 |
| unmatchedRows | 33 |
| matchRate | 0.6071 |
| total outcomes (recon + noDecision) | 94 |
| registryRows | 27 |
| liveRows | 1 |
| fallbackRows | 1 |

`/rows` returned 263. The 10 remaining backlog gamePks map to 11 active rows (824818 has 2) -- all 11 still
`outcomeStatus = null` (pending). Both endpoints functional.

## delta metrics

All zero this slice -- no outcome settled: total outcomes +0, reconciledRows +0, noDecisionRows +0, matchedRows
+0, unmatchedRows +0, matchRate unchanged (0.6071). No per-route or per-regime movement.

Pending shape on eventual settlement (unchanged from v4's deferred set): the 10 remaining runs are all
no-decision (starter_missing) EXCEPT the three targeted `starter_enriched_market_missing` runs (823442, 823765,
823119) which are directional (all lean home) and will yield the **first directional read on the
`enriched_market_missing` route** once Final. 824818 (28bd433e) and the six targeted `starter_missing_market_
missing` games settle as noDecisionRows (coverage/abstention).

## calibration notes

**None** -- no directional or no-decision backlog outcome settled this slice, so there is no confidence-vs-
matchedOutcome, analyzerConfidence-vs-calibrated, evidenceRichness-vs-outcome, enriched_market_backed_depth,
enriched_market_missing, or no-decision-coverage movement. Deferred to the first pass that reconciles a Final
game. The v4 read stands as the current calibration state (`calibration-delta-v1.md`).

## tests / checks run (non-paid)

- Free StatsAPI settlement probe for all 10 remaining gamePks (status + scores).
- Read-only `/metrics` + `/rows` before/after (identical). 824818 collision re-check via `/rows`.
- No code changed -> no test suite required.

## paid-call status

**None.**

## buyer-facing impact

**None.** No buyer UX, copy, body, routes, or pricing touched.

## default allowlist status

**Unchanged** -- exactly four regimes in `DEFAULT_ALLOWLIST`.

## files changed

Vault only: this doc + the handoff entry. `dai` HEAD unchanged at `1a2ddf2`. **Prompts/registry,
DEFAULT_ALLOWLIST, buyer copy, DB schema, model prompts: untouched.**

## risks / deferred items

- Time-gated no-op: the remaining backlog is blocked until the games complete. 824818 (07-01) is in `Preview`
  (first pitch not reached); the 07-02 slate is future. The real-world clock has not advanced past these games
  since v4.
- **824818 requires per-run reconciliation** (`/{28bd433e}/outcome`) due to the `3ade423e` collision -- do NOT
  use identity `/reconcile` for it.
- Gate the next attempt strictly on StatsAPI status flipping to `Final` (824818 first) to avoid another no-op.
- API + devcore-sql left running at slice end; stop when no longer needed.

## next recommended slice

**Outcome Reconciliation Follow-up v6** once StatsAPI shows 824818 (07-01) `Final` -- per-run reconcile on
`28bd433e` (MultipleMatches expected), then the nine 07-02 targeted games as they finalize (first directional
read on the `enriched_market_missing` route via 823442 / 823765 / 823119). A non-time-gated alternative that
makes progress now: **Registry Assembly Error Diagnostic v1** (still open from the readiness slice).
