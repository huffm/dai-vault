# Reconcile 7 Usable MLB Stage 0 Runs v1

**date:** 2026-06-15
**status:** wait-only -- blocked on settlement. all 7 target games are pre-start (`Preview`/`Scheduled`) at execution time, so no trusted final exists and no reconciliation was performed. no `POST /api/agent-runs/reconcile` payload was submitted, no outcome was fabricated, no non-final probe was posted. no code, no spend.
**classification:** pre-start wait-only; no calibration evidence produced.

## Why wait-only

The 7 runs were generated this afternoon for games that play tonight UTC. At execution the clock is before first pitch for every one of them.

- current UTC (system + DB): `2026-06-15 18:18Z`.
- earliest target start: `2026-06-15 22:40Z` (Marlins @ Phillies) -- ~4h22m away.
- latest target start: `2026-06-16 02:10Z` (Rays @ Dodgers).
- StatsAPI `abstractGameState` for all 7 gamePks: **Preview / Scheduled**. None is Final; linescore runs are null.

Per the reconciliation boundary ("confirm the game is Final via StatsAPI before reconciling") and the no-fabrication rule, reconciliation cannot proceed. Posting a payload now would either fabricate a final or register a synthetic non-final test -- both forbidden.

## Pre / post counts (unchanged)

| | outcomes | evaluations |
|---|---|---|
| before | 12 | 12 |
| after | 12 | 12 |

No outcome or evaluation row was written. The four prior reconciled candidates are untouched.

## 7-run reconciliation table (all pending)

| run id (8) | matchup (away @ home) | gamePk | AgentRun.LeanSide | StatsAPI state | final score | winning side | evaluation | matcher status |
|---|---|---|---|---|---|---|---|---|
| `2303433e` | Marlins @ Phillies | 823452 | home | Preview/Scheduled | pending | pending | pending | not attempted (pre-start) |
| `2803433e` | Royals @ Nationals | 822724 | home | Preview/Scheduled | pending | pending | pending | not attempted (pre-start) |
| `2a03433e` | Mets @ Reds | 824505 | home | Preview/Scheduled | pending | pending | pending | not attempted (pre-start) |
| `3403433e` | Rockies @ Cubs | 824666 | home | Preview/Scheduled | pending | pending | pending | not attempted (pre-start) |
| `3d03433e` | Tigers @ Astros | 824181 | home | Preview/Scheduled | pending | pending | pending | not attempted (pre-start) |
| `4103433e` | Angels @ Diamondbacks | 825071 | home | Preview/Scheduled | pending | pending | pending | not attempted (pre-start) |
| `4903433e` | Rays @ Dodgers | 823938 | home | Preview/Scheduled | pending | pending | pending | not attempted (pre-start) |

Identity was confirmed earlier (Budgeted MLB Stage 0 Generation v1): all 7 carry `mlb_statsapi` + the exact gamePk above, under `TenantKey=1`. No identity/tenant/source/matcher issue surfaced -- only timing.

## Correct / incorrect tally

correct 0 / incorrect 0 / inconclusive 0 (none reconciled this slice).

## Directionally usable reconciled sample after this slice

Unchanged. The reconciled directional sample remains the 3 from the first reconciliation slice (2 correct / 1 incorrect; the 4th was an inconclusive null-lean). These 7 will add up to 7 more directional samples once settled and reconciled (total would reach ~10 directional). The sample **remains too small for any confidence/posture threshold change** (ledger entry 12 stays gated) regardless.

## Null-lean exclusion

The 3 null-lean runs (`2e03433e` Padres @ Cardinals, `3603433e` Twins @ Rangers, `4203433e` Pirates @ Athletics) were intentionally excluded from directional calibration, per the slice and Directional Lean Contract v1 (they are true non-evaluable abstentions and would evaluate inconclusive).

## Recommended next slice

Rerun this exact reconciliation after settlement. The latest target starts at `2026-06-16 02:10Z`; allowing ~3h for a 9-inning game, all 7 should be Final by roughly `2026-06-16 05:30Z`. After that, reconcile the 7 exact `(mlb_statsapi, gamePk)` keys through `POST /api/agent-runs/reconcile` with StatsAPI finals (`source=manual:statsapi`, `sourceRef` the feed URL), expecting `SingleMatch` + correct/incorrect for each, count +7, and the 409 guard on a re-post. No code, no model spend.
