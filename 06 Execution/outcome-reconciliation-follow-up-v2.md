# Outcome Reconciliation Follow-up v2 -- Evidence Report

**status:** complete (20-run backlog re-checked; 0 Final; 0 reconciled; 20 deferred -- still time-gated)
**date:** 2026-06-30

## purpose

Re-attempt reconciliation of the pending 20-run live backlog (3 soak + 8 v2 batch + 9 targeted), reconcile only
Final games, and produce an evidence-backed summary. Non-paid; settlement-gated; no fabricated outcomes. Second
attempt after Outcome Reconciliation Follow-up v1 found all 20 pending.

## start state

- `dai`: clean, synced, `d1fbb33` (0/0). `dai-vault`: clean, synced, `f4aba65` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4). devcore-sql up. No paid calls.

## backlog set (20 runs, all provenance-bearing, all unreconciled at start)

DB confirmed: `BacklogReconciled = 0`; total runs 263, total outcomes 84 (the prior reconciled set, unchanged).

| group | keys | runs | regime(s) | lean |
|---|---|---|---|---|
| soak | 260013-260015 | 3 | starter_missing_market_backed_depth (1), starter_missing_market_missing (2) | all null (no-decision) |
| v2 batch | 270013-270020 | 8 | starter_enriched_market_backed_depth | all directional (5 home / 3 away) |
| targeted | 270021-270026 | 6 | starter_missing_market_missing | all null (no-decision) |
| targeted | 270027-270029 | 3 | starter_enriched_market_missing | all directional (home) |

## settlement status (re-probed live via free StatsAPI)

`statsapi.mlb.com/api/v1/schedule?gamePk=...&hydrate=linescore` for all 20 gamePks:

- **0 Final.** 15 games `Scheduled`; **5 of the 06-30 slate now `Pre-Game`** (824907, 824096, 824175, 823528,
  822793 -- first pitch imminent, 0-0). No scores posted on any game.
- 06-30 slate (10 games) not yet complete (5 scheduled + 5 pre-game). 824818 (07-01) and the 9 targeted (07-02)
  remain future.

The clock advanced slightly vs Follow-up v1 (5 games moved Scheduled -> Pre-Game), but none have reached Final.

## reconciliation actions

**None.** 0 of 20 games are Final -> 0 eligible. Per the rules ("reconcile only Final"), no `/reconcile` or
per-run `/outcome` calls were made. No outcomes fabricated, no non-final game reconciled.

## runs reviewed / reconciled / deferred

- **Reviewed: 20.** **Reconciled: 0.** **Deferred: 20** (reason: not Final -- games unplayed/in-progress-to-start).

Deferred breakdown by eventual outcome shape (once Final):
- **11 directional** -> will yield matched/unmatched: 8 starter_enriched_market_backed_depth (v2) + 3
  starter_enriched_market_missing (targeted).
- **9 no-decision** (null lean) -> will become noDecisionRows, not a directional hit: 1
  starter_missing_market_backed_depth + 8 starter_missing_market_missing (soak + targeted). Their calibration
  value is coverage/abstention behavior, not win rate.

## aggregate reconciliation metrics (unchanged -- nothing settled)

No outcome was written, so the calibration totals are identical to the prior snapshot (verified via DB: backlog
reconciled = 0, total outcomes = 84):

| metric | value |
|---|---|
| totalRows | 263 |
| reconciledRows | 76 |
| unreconciledRows | 179 |
| matchedRows | 47 |
| unmatchedRows | 29 |
| noDecisionRows | 8 |
| matchRate | 0.6184 |
| registryRows | 27 |
| liveRows | 1 |
| fallbackRows | 1 |

(The 47/29 matched/unmatched and 0.6184 match rate are from the historical/earlier-reconciled set, not the
backlog -- the backlog contributes nothing until it settles.)

## calibration signals

**None assessable this slice** -- a directional calibration read (overconfidence, weak-evidence-too-strong,
market-following, disagreement outcomes, shallow-source failure) requires reconciled directional outcomes, and
the backlog produced zero. Deferred to the pass that reconciles the 11 directional runs. The 9 no-decision runs
will never produce a directional calibration signal by design (starter_missing -> abstention).

## tests / checks run (non-paid)

- DB: backlog-reconciled query (0), regime/lean breakdown, totals.
- Free StatsAPI settlement probe for all 20 gamePks (status + scores).
- No API restart (nothing to reconcile). No code changed -> no test suite required.

## paid-call status

**None.** StatsAPI reads are the free public endpoint; DB reads only. No model call.

## files changed

Vault only: this doc + the handoff entry. **No dai code, no prompt/registry, no DEFAULT_ALLOWLIST, no buyer copy,
no DB schema** touched (all verified untouched -- dai HEAD unchanged at d1fbb33).

## confirmation of untouched surfaces

- Prompts / prompt registry: **untouched.**
- DEFAULT_ALLOWLIST: **unchanged** (4 regimes).
- Buyer copy / buyer routes: **untouched.**
- DB schema: **untouched** (no migration).

## risks / deferred items

- The entire 20-run backlog remains pending; reconciliation is fully blocked until the games are played. The
  06-30 slate is at first pitch (Pre-Game) and should settle within hours; 824818 (07-01) and the 07-02 targeted
  games settle later. Gate the next attempt on StatsAPI status flipping to `Final`.
- This is the second consecutive time-gated no-op reconciliation; the work is event-driven, not within our
  control. Re-run when the 06-30 games complete (likely imminent).
- The 9 no-decision starter_missing runs add coverage but no match rate on settlement.

## next recommended slice

**Outcome Reconciliation Follow-up v3** -- re-run within hours once the 06-30 slate reaches Final (5 are already
Pre-Game), reconciling that subset and producing the first directional calibration read on the v2
enriched_market_backed_depth runs. If a fully non-time-gated slice is preferred meanwhile,
**Calibration Metrics Export Download v1** is pure read/report work (export the current per-route metrics for
offline analysis) with no settlement dependency.
