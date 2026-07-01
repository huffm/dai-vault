---
title: "Outcome Reconciliation Follow-up v1"
type: "reconciliation"
date: "2026-06-30"
status: "no-op"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - outcome
related:
  - "06 Execution/outcome-reconciliation-readiness-next-slice-2026-06-30.md"
---

# Outcome Reconciliation Follow-up v1 -- Evidence Report

**status:** complete (all 20 backlog games checked; 0 Final; 0 reconciled; 20 remain PENDING -- time-gated)
**date:** 2026-06-30

## purpose

Check the 20-run live reconciliation backlog (3 soak + 8 v2 batch + 9 targeted), reconcile only games that are
Final, and refresh prompt-route calibration metrics so live route coverage becomes performance evidence.
Non-paid; settlement-gated; no fabricated outcomes.

## start state

- `dai`: clean, synced, `0f563d6` (0/0). `dai-vault`: clean, synced, `2719a45` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4). Route fix v1 present (line 147). Taxonomy split NOT implemented
  (no `starter_complete`/`starter_asymmetric` in source). devcore-sql up.

## target run table (20 runs, all provenance-bearing, all unreconciled at start)

| group | key | runId | gamePk | away @ home | gameDate | regime | lean |
|---|---|---|---|---|---|---|---|
| soak | 260013 | 1fbd433e | 824338 | Marlins @ Rockies | 2026-06-30 | starter_missing_market_backed_depth | null |
| soak | 260014 | 25bd433e | 825066 | Giants @ Dbacks | 2026-06-30 | starter_missing_market_missing | null |
| soak | 260015 | 28bd433e | 824818 | White Sox @ Orioles | 2026-07-01 | starter_missing_market_missing | null |
| v2 | 270013 | c240433e | 824907 | Cardinals @ Braves | 2026-06-30 | starter_enriched_market_backed_depth | home |
| v2 | 270014 | c440433e | 824096 | Rays @ Royals | 2026-06-30 | starter_enriched_market_backed_depth | away |
| v2 | 270015 | ca40433e | 824175 | Twins @ Astros | 2026-06-30 | starter_enriched_market_backed_depth | away |
| v2 | 270016 | cf40433e | 824984 | Dodgers @ Athletics | 2026-06-30 | starter_enriched_market_backed_depth | away |
| v2 | 270017 | d040433e | 823122 | Angels @ Mariners | 2026-06-30 | starter_enriched_market_backed_depth | home |
| v2 | 270018 | d640433e | 824661 | Padres @ Cubs | 2026-06-30 | starter_enriched_market_backed_depth | home |
| v2 | 270019 | db40433e | 823528 | Tigers @ Yankees | 2026-06-30 | starter_enriched_market_backed_depth | home |
| v2 | 270020 | de40433e | 822793 | Mets @ Blue Jays | 2026-06-30 | starter_enriched_market_backed_depth | home |
| targeted | 270021 | e440433e | 824335 | Marlins @ Rockies | 2026-07-02 | starter_missing_market_missing | null |
| targeted | 270022 | e540433e | 824416 | White Sox @ Guardians | 2026-07-02 | starter_missing_market_missing | null |
| targeted | 270023 | ea40433e | 824906 | Cardinals @ Braves | 2026-07-02 | starter_missing_market_missing | null |
| targeted | 270024 | eb40433e | 824093 | Rays @ Royals | 2026-07-02 | starter_missing_market_missing | null |
| targeted | 270025 | ef40433e | 822884 | Tigers @ Rangers | 2026-07-02 | starter_missing_market_missing | null |
| targeted | 270026 | f640433e | 823935 | Padres @ Dodgers | 2026-07-02 | starter_missing_market_missing | null |
| targeted | 270027 | f940433e | 823442 | Pirates @ Phillies | 2026-07-02 | starter_enriched_market_missing | home |
| targeted | 270028 | fc40433e | 823765 | Reds @ Brewers | 2026-07-02 | starter_enriched_market_missing | home |
| targeted | 270029 | 0341433e | 823119 | Angels @ Mariners | 2026-07-02 | starter_enriched_market_missing | home |

All 20 confirmed: provenance persisted, `ExclusionReason = null` (active), 0 `AgentRunOutcome` rows at start.

## settlement status by run

Checked live via `statsapi.mlb.com/api/v1/schedule?gamePk=...&hydrate=linescore` (free, non-paid). **All 20
games are `Scheduled` / abstractGameState `Preview` -- none Final.** No scores posted.

- 06-30 slate (10 games: 824338, 825066, 824907, 824096, 824175, 824984, 823122, 824661, 823528, 822793) --
  not yet started (first pitch 06-30 22:36 UTC onward; current time precedes it). PENDING.
- 07-01 game (824818) -- future. PENDING.
- 07-02 slate (9 games: 824335, 824416, 824906, 824093, 822884, 823935, 823442, 823765, 823119) -- future.
  PENDING.

## reconciliation actions

**None.** 0 of 20 games are Final, so 0 are eligible. Per the reconciliation rules ("If not Final, leave
pending"), no `/reconcile` or per-run `/outcome` calls were made. No outcomes fabricated, no future game
reconciled, no MultipleMatches situation reached.

## final scores used

None -- no game has a final score yet.

## AgentRunOutcome + paired evaluation evidence

Backlog reconciled count verified in DB = **0**. `AgentRunOutcomes` total remains 84 (the prior reconciled set
from the earlier readiness slice: 7 registry + 1 assembly_error in the enriched_backed_depth regime, plus 76
legacy/historical). No new outcome or evaluation rows written this slice.

## metrics before / after (unchanged -- nothing reconciled)

Verified via DB (no metric depends on a change this slice): totalRows 263, backlog-reconciled 0, route counts
identical to the prior snapshot.

| metric | before | after |
|---|---|---|
| totalRows | 263 | 263 |
| reconciledRows | 76 | 76 |
| unreconciledRows | 179 | 179 |
| matchedRows | 47 | 47 |
| unmatchedRows | 29 | 29 |
| noDecisionRows | 8 | 8 |
| matchRate | 0.6184 | 0.6184 |
| registryRows | 27 | 27 |
| liveRows | 1 | 1 |
| fallbackRows | 1 | 1 |
| unknownRouteRows | 235 | 235 |

## route-level results (current state, unchanged)

| route | runs | reconciled |
|---|---|---|
| starter_enriched_market_backed_depth (registry) | 15 | 7 |
| starter_enriched_market_backed_depth::assembly_error (live) | 1 | 1 |
| starter_enriched_market_missing (registry) | 3 | 0 |
| starter_missing_market_missing (registry) | 8 | 0 |
| starter_missing_market_backed_depth (registry) | 1 | 0 |

The 17 recently captured registry routes (v2 + targeted) remain 0-reconciled because their games are unplayed.

## no-decision analysis

9 of the 20 backlog runs are no-decision (null lean): the 3 soak + 6 targeted starter_missing_* runs. When these
games go Final and reconcile, they will become `noDecisionRows` (reconciled outcome but no directional lean) --
not matched/unmatched. The 11 directional runs (8 enriched_backed_depth + 3 enriched_market_missing) will produce
matched/unmatched once settled. This slice changed nothing (no game Final), but documents the expected split for
the next pass.

## fallback route-key verification

The fallback route `starter_enriched_market_backed_depth::assembly_error` remains present and correctly
attributed (1 run, src=live, reconciled), NOT collapsed to `unknown`. Calibration Route Attribution Fix v1
holds.

## paid-call status

**None.** No model calls. StatsAPI schedule reads are the free public endpoint; DB reads only.

## buyer-facing impact

**None.**

## risks / deferred items

- Entire 20-run backlog remains pending -- reconciliation is fully blocked until the games are played. Settlement
  windows: 06-30 games settle late 06-30 / early 07-01; 824818 on 07-01; 07-02 games on 07-02/07-03.
- Re-running this slice prematurely (before settlement) will again find 0 Final -- gate the next run on the
  StatsAPI status flipping to Final.
- When the starter_missing_* games settle they add noDecisionRows, not match rate -- their calibration value is
  coverage/abstention behavior (per the taxonomy plan), so do not expect a directional read from them.
- Point-in-time snapshot (2026-06-30); status will change as games are played.

## next recommended slice

**Continue Outcome Reconciliation Follow-up v1** -- re-run this exact reconciliation once the games reach Final
(start with the 06-30 slate after it completes, then 07-01, then 07-02). It is the prerequisite for converting
the new route coverage into performance evidence and for any allowlist-promotion decision.

Because reconciliation is time-blocked, a productive non-blocked alternative is **Asymmetric-Enriched Recipe +
Regime Split v1** (the implementation slice from the taxonomy plan) -- the enriched_backed_depth regime already
has 8 reconciled rows to anchor it, and the asymmetric split is justified by the deterministic assembly_error
failure independent of the pending backlog. Sequence per operator preference: reconcile-when-settled is the
cleaner evidence path; the split is the way to make code progress while the games are unplayed.
