---
title: "Outcome Reconciliation Follow-up v6"
type: "reconciliation"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v6"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - outcome
related:
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v5.md"
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v4.md"
---

# Outcome Reconciliation Follow-up v6 -- Evidence Report

## status

Complete. gamePk 824818 (the 07-01 soak game) reached Final; reconciled the one backlog run via per-run
`/outcome` (MultipleMatches collision avoided). 1 of the remaining 10 backlog runs settled; 9 still pending
(the nine 07-02 targeted games). Non-paid; settlement-gated; scores verbatim from StatsAPI; no directional
movement (the run is a no-decision starter_missing soak).

## start state (verified)

- `dai`: clean, synced, `b2f9771` (0/0), unchanged this slice. `dai-vault`: clean, synced (OKF backfill v1-v6
  pushed). Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4). No code/prompt/schema change. Platform API up on `:5007` (`/health` ok),
  `devcore-sql` up. FastAPI agent-service NOT started (reconciliation is a platform-only path, no model call).

## settlement probe (free StatsAPI)

`statsapi.mlb.com/api/v1/schedule?sportId=1&gamePks=824818&hydrate=linescore` (free public endpoint -- not a paid
call). **824818 White Sox @ Orioles (2026-07-01): detailedState=Game Over, abstractGameState=Final, away 1 /
home 6 -> home_win.** Gate: >=1 backlog game Final -> proceed with v6.

## identity-collision handling (per-run only, as pre-planned)

gamePk 824818 has **2 active runs** (`/rows`, both `ExclusionReason IS NULL`, both lean-null):

| agentRunId | role | regime | pre-state |
|---|---|---|---|
| `3ade423e-f36b-1410-816d-00373db4b724` | non-backlog | (none) | outcome null |
| `28bd433e-f36b-1410-816e-00373db4b724` | backlog soak (key 260015) | starter_missing_market_missing | outcome null |

Identity `/reconcile` would return `MultipleMatches` and write nothing. As pre-planned across v4/v5, reconciled
**only** the backlog run `28bd433e` via per-run `/{id}/outcome`. **Identity `/reconcile` was NOT used for 824818.**

## reconciliation action (through the real API guards)

- **Per-run `POST /api/agent-runs/28bd433e-.../outcome`:** `home_win`, homeScore 6, awayScore 1,
  source `statsapi_final`, sourceRef `gamePk:824818`, note recording the collision. Wrote outcome + paired
  evaluation. **0 integrity refusals (422).**
- Result: `outcomeStatus=home_win`, `resultSide=home`. Because the run's persisted `leanSide` is null
  (starter_missing = no-decision), the evaluation is inconclusive -> a **noDecisionRow**, not matched/unmatched.

## non-backlog run left untouched (verified after write)

`/rows` after the write: `3ade423e` -> `outcomeStatus = null` (UNTOUCHED). `28bd433e` -> `outcomeStatus =
home_win`, `resultSide = home`. The per-run write targeted exactly the backlog run; the non-backlog run keeps
zero outcome rows and will keep producing `MultipleMatches` on any identity reconcile for 824818 until it is
excluded or reconciled on its own (out of scope).

## metrics delta (`/prompt-route-calibration/metrics`, tenant-scoped)

| metric | before | after | delta |
|---|---|---|---|
| totalRows | 263 | 263 | 0 |
| reconciledRows (directional) | 84 | 84 | 0 |
| noDecisionRows | 10 | 11 | +1 |
| total outcomes (recon + noDecision) | 94 | 95 | +1 |
| matchedRows | 51 | 51 | 0 |
| unmatchedRows | 33 | 33 | 0 |
| matchRate | 0.6071 | 0.6071 | 0 |
| registryRows / liveRows / fallbackRows | 27 / 1 / 1 | 27 / 1 / 1 | 0 |

Route-level: `starter_missing_market_missing` (registry) noDecisionRows **1 -> 2** (the 825066 soak from v4 plus
this 824818 soak). No matchRate change anywhere -- a no-decision settlement adds coverage, not a directional hit.

## backlog status

- **Reconciled: 11 of 20** (10 in Follow-up v4 + 1 this slice).
- **Pending: 9** -- the nine 07-02 targeted games (824335, 824416, 824906, 824093, 822884, 823935, 823442,
  823765, 823119). Of these, 3 are directional `starter_enriched_market_missing` (823442, 823765, 823119, all
  lean home) -> will give the **first directional read on the enriched_market_missing route** once Final; 6 are
  no-decision `starter_missing_market_missing`.

## calibration notes

**None new.** 824818 is a no-decision soak run, so it produces no directional / confidence / market signal --
only abstention coverage. The current calibration read (Calibration Delta v1) is unchanged. The next directional
movement comes when the 07-02 `enriched_market_missing` runs settle.

## tests / checks run (non-paid)

- Free StatsAPI settlement probe for 824818 (status + final score).
- `/rows` before/after (collision pre-check + non-backlog-untouched confirmation); `/metrics` before/after.
- No code changed -> no test suite required.

## paid-call status

**None.** StatsAPI read is the free public endpoint; the reconcile went through the platform DB/API only.

## buyer-facing impact / default allowlist

**None.** `DEFAULT_ALLOWLIST` unchanged (4). No buyer UX/copy/routes/pricing.

## files changed

Vault only: this doc + the handoff entry. `dai` HEAD unchanged at `b2f9771`. **Prompts/registry,
DEFAULT_ALLOWLIST, buyer copy, DB schema, model prompts: untouched.**

## risks / deferred items

- 9 backlog runs remain pending (07-02 targeted); gate the next attempt on StatsAPI Final. Check for gamePk
  collisions again before each settle (use per-run `/outcome` on any MultipleMatches).
- `3ade423e` (824818 non-backlog) stays unreconciled -> keeps producing `MultipleMatches` for 824818 until
  excluded/reconciled. Left as-is.
- API + devcore-sql left running; stop when no longer needed.

## next recommended slice

**Outcome Reconciliation Follow-up v7** once any 07-02 game is Final -- reconcile the SingleMatch games via
identity `/reconcile`, and use per-run `/outcome` for any collision. The three `enriched_market_missing`
directional runs are the highest-value settle (first directional read on that route). Non-time-gated alternative:
**OKF Backfill Batch v7** (finish the vault migration).
