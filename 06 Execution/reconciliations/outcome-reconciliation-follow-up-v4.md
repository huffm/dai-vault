---
title: "Outcome Reconciliation Follow-up v4"
type: "reconciliation"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v4 + Calibration Delta v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - outcome
  - calibration
  - metrics
related:
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v3.md"
  - "06 Execution/outcome-reconciliation-readiness-next-slice-2026-06-30.md"
  - "04 Products/sports-v1/calibration/calibration-delta-v1.md"
---

# Outcome Reconciliation Follow-up v4 -- Evidence Report

## status

Complete. **First non-no-op reconciliation of the 20-run backlog:** the 2026-06-30 slate went Final overnight;
10 of the 20 backlog runs reconciled through the real API guards. 8 directional (4 correct / 4 incorrect = 0.50)
+ 2 no-decision (inconclusive). 10 backlog runs remain pending (07-01 + 07-02). Non-paid; settlement-gated; no
fabricated outcomes; scores verbatim from StatsAPI.

## start state (verified)

- `dai`: clean, synced, `1a2ddf2` (0/0). `dai-vault`: clean, synced, `26469bc` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4 regimes). No code/prompt/schema change this slice.
- `devcore-sql` docker container was `Exited (255)` after a Docker Desktop stop; `docker start devcore-sql`
  brought it up healthy on `:1433`. Platform API built (0 warn / 0 err) and run on `:5007`
  (`ASPNETCORE_ENVIRONMENT=Development`, dev bypass auth, TenantKey=1). `/health` -> ok. FastAPI agent-service
  NOT started (reconciliation is a platform-only path, no model call).

## settlement probe (free StatsAPI, all 20 backlog gamePks)

`statsapi.mlb.com/api/v1/schedule?sportId=1&gamePks=...&hydrate=linescore` (free public schedule endpoint --
not a paid model call, no analysis run). Result: **10 Final, 10 non-Final.**

- **Final (10) -- the entire 06-30 slate:** 822793, 823122, 823528, 824096, 824175, 824338, 824661, 824907,
  824984, 825066.
- **Non-Final (10):** 824818 (07-01, Scheduled) + the nine 07-02 targeted games (822884, 823119, 823442,
  823765, 823935, 824093, 824335, 824416, 824906) all Scheduled / Preview. No scores posted.

Decision gate: >=1 backlog game Final -> proceed to Follow-up v4 + Calibration Delta v1 (this slice).

## final scores used (verbatim from StatsAPI, away @ home)

| gamePk | game | away | home | winner |
|---|---|---|---|---|
| 822793 | Mets @ Blue Jays | 3 | 0 | away_win |
| 823122 | Angels @ Mariners | 3 | 8 | home_win |
| 823528 | Tigers @ Yankees | 9 | 3 | away_win |
| 824096 | Rays @ Royals | 10 | 4 | away_win |
| 824175 | Twins @ Astros | 4 | 6 | home_win |
| 824338 | Marlins @ Rockies | 14 | 3 | away_win |
| 824661 | Padres @ Cubs | 7 | 9 | home_win |
| 824907 | Cardinals @ Braves | 5 | 3 | away_win |
| 824984 | Dodgers @ Athletics | 9 | 3 | away_win |
| 825066 | Giants @ Dbacks | 2 | 8 | home_win |

## identity-collision analysis (before any write)

Pulled `/api/agent-runs/prompt-route-calibration/rows` (active runs only -- `ExclusionReason IS NULL`) and
counted active runs per Final gamePk:

- **9 gamePks -> single active run** -> safe for identity `/reconcile` (SingleMatch).
- **825066 -> 2 active runs** (`37de423e` non-backlog + backlog soak `25bd433e`, both lean-null). Identity
  `/reconcile` would return `MultipleMatches` and write nothing. Reconciled **only** the backlog run
  `25bd433e` via per-run `/{id}/outcome`; left `37de423e` untouched (out of scope). Same doctrine as the prior
  824662 collision (Follow-up readiness slice).

## reconciliation actions (all through real API guards)

All writes went through the settlement direction-integrity guard + `RunEvaluator` via the platform API. No
direct-SQL outcome writes, no fabricated scores. `sourceProvider = mlb_statsapi`, `source = statsapi_final`,
`sourceRef = gamePk:<pk>`.

- **9 identity-driven `/reconcile` (all returned SingleMatch, each recorded outcome + paired evaluation):**
  822793, 823122, 823528, 824096, 824175, 824338, 824661, 824907, 824984.
- **1 per-run `/{id}/outcome` (825066 backlog soak `25bd433e`):** home_win 8-2, note recording the
  identity collision. Targets exactly the backlog run without touching `37de423e`.
- **0 integrity refusals (422).** No run's persisted lean contradicted its artifact prose.
- The 10 non-Final backlog runs were left untouched (not Final). No non-final game reconciled.

## per-run results

| gamePk | game | route | lean | conf | outcome | eval |
|---|---|---|---|---|---|---|
| 822793 | Mets @ Blue Jays | enriched_market_backed_depth | home | 0.75 | away_win | **incorrect** |
| 823122 | Angels @ Mariners | enriched_market_backed_depth | home | 0.75 | home_win | **correct** |
| 823528 | Tigers @ Yankees | enriched_market_backed_depth | home | 0.75 | away_win | **incorrect** |
| 824096 | Rays @ Royals | enriched_market_backed_depth | away | 0.75 | away_win | **correct** |
| 824175 | Twins @ Astros | enriched_market_backed_depth | away | 0.75 | home_win | **incorrect** |
| 824661 | Padres @ Cubs | enriched_market_backed_depth | home | 0.75 | home_win | **correct** |
| 824907 | Cardinals @ Braves | enriched_market_backed_depth | home | 0.80 | away_win | **incorrect** |
| 824984 | Dodgers @ Athletics | enriched_market_backed_depth | away | 0.80 | away_win | **correct** |
| 824338 | Marlins @ Rockies | starter_missing_market_backed_depth | null | -- | away_win | inconclusive (no-decision) |
| 825066 | Giants @ Dbacks | starter_missing_market_missing | null | -- | home_win | inconclusive (no-decision) |

**Directional record: 4 correct / 4 incorrect = 0.500** (all 8 in the enriched_market_backed_depth registry
route). 2 no-decision runs reconciled as inconclusive (coverage/abstention, not win rate). Full calibration read
in `calibration-delta-v1.md`.

## metrics delta (`/prompt-route-calibration/metrics`, tenant-scoped)

| metric | before (v3 close) | after (v4) | delta |
|---|---|---|---|
| totalRows | 263 | 263 | 0 |
| reconciledRows (directional) | 76 | 84 | +8 |
| noDecisionRows | 8 | 10 | +2 |
| total outcomes (recon+noDecision) | 84 | 94 | +10 |
| unreconciledRows | 179 | 169 | -10 |
| matchedRows | 47 | 51 | +4 |
| unmatchedRows | 29 | 33 | +4 |
| matchRate | 0.6184 | 0.6071 | -0.0113 |
| registryRows | 27 | 27 | 0 |
| liveRows | 1 | 1 | 0 |
| fallbackRows | 1 | 1 | 0 |

`+10` total outcomes = exactly the 10 games reconciled (8 directional + 2 no-decision). Aggregate matchRate
dips because the 0.50 v2 batch dilutes the historical 0.6184.

### route-level after

- `starter_enriched_market_backed_depth` (registry): total 15, reconciled **15/15**, matched **10**,
  unmatched **5**, matchRate **0.667** (was 7 reconciled / matched 6 / unmatched 1 / 0.857). confMatched 0.745,
  confUnmatched 0.760 -- the wrong calls carried *higher* average confidence (non-predictive signal persists).
- `starter_missing_market_backed_depth` (registry): total 1, noDecisionRows **1** (824338 soak).
- `starter_missing_market_missing` (registry): total 8, noDecisionRows **1** (825066 backlog soak; 7 targeted
  still pending).
- `starter_enriched_market_missing` (registry): total 3, reconciled 0 (all 3 pending -- 07-02).

## backlog verification (before == expected after)

`/rows` after-state, backlog set: **10 reconciled, 10 pending.** Pending = soak 824818 (07-01) + nine 07-02
targeted. Recomputed directional from rows (leanSide vs resultSide): 8 directional / 4 correct / 4 incorrect;
2 no-decision. Consistent with the metric deltas.

## tests / checks run (non-paid)

- Free StatsAPI settlement probe for all 20 gamePks (status + final scores).
- Collision pre-check via `/rows` (active runs per gamePk).
- Metrics before/after + `/rows` before/after cross-check. No code changed -> no test suite required.

## paid-call status

**None.** No model calls. StatsAPI schedule reads are the free public endpoint. All reconciliation went
through the platform DB/API only.

## buyer-facing impact

**None.** Internal calibration/reconciliation only. No buyer UX, copy, body, routes, or pricing touched.

## default allowlist status

**Unchanged** -- exactly four regimes in `DEFAULT_ALLOWLIST`.

## files changed

Vault only: this doc + `04 Products/sports-v1/calibration/calibration-delta-v1.md` + the handoff entry.
`dai` HEAD unchanged at `1a2ddf2`. **Prompts/registry, DEFAULT_ALLOWLIST, buyer copy, DB schema: untouched.**

## risks / deferred items

- **10 backlog runs still pending:** soak 824818 (07-01) + nine 07-02 targeted. Gate the next attempt on
  StatsAPI status flipping to `Final` (824818 first). **824818 shares its gamePk with a non-backlog active run
  (`3ade423e` + backlog `28bd433e`)** -> will return `MultipleMatches` on identity reconcile; use per-run
  `/{28bd433e}/outcome` when it settles, mirroring 825066 this slice.
- 825066's non-backlog run `37de423e` stays unreconciled and will keep producing `MultipleMatches` on any
  identity-driven reconcile for that game until excluded or reconciled on its own. Left as-is (out of scope).
- The `enriched_market_backed_depth` read is now n=15 reconciled but only n=8 fresh this slice -- still under
  Gate-4 sufficiency. Do not tune (see calibration-delta-v1.md).
- API + devcore-sql left running at slice end for any immediate follow-up; stop when no longer needed.

## next recommended slice

**Outcome Reconciliation Follow-up v5** once StatsAPI shows 824818 (07-01) Final -- per-run reconcile on
`28bd433e` (MultipleMatches expected). Then the nine 07-02 targeted games as they finalize. Each populates the
two `starter_missing` no-decision routes and (for the 3 enriched_market_missing) the first directional read on
that route. A non-time-gated alternative: the **Registry Assembly Error Diagnostic v1** (still open from the
readiness slice).
