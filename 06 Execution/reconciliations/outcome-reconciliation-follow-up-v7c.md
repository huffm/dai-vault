---
title: "Outcome Reconciliation Follow-up v7c"
type: "reconciliation"
date: "2026-07-03"
status: "complete-verification-only"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v7c"
repos:
  dai: "unchanged (6c13b1d)"
  dai-vault: "docs-only"
tags:
  - reconciliation
  - outcome
  - calibration
related:
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v7b.md"
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v7.md"
---

# Outcome Reconciliation Follow-up v7c (no new writes: backlog was already auto-settled; verified + reassessed)

**slice:** settle the remaining seven 07-02 backlog games when Final; rerun pooled reassessment
**status:** complete 2026-07-03 ~16:04Z. **zero reconciliation writes by this pass** -- all 7 were
already settled by an out-of-process poller burst at 2026-07-03T15:39:13-16Z. this pass verified them
against source of truth, ran the pooled reassessment, and analysed residue. no tuning.
**repos:** `dai` unchanged (`6c13b1d`, phantom-M on DevCore.Data.csproj is byte-identical, ignored).
`dai-vault` docs-only (this doc + slice entry).

## corrected slice identity (stale-state note)

this is **v7c**, not v7b. two layers of stale state were corrected:

1. the *original* handoff was stale (already flagged by the operator).
2. the operator's *corrected* handoff was **also** partially stale: it stated "no v7c reconciliation
   writes have happened yet" and framed 7 games as unreconciled candidates to settle. the DB shows all
   7 were already settled at **2026-07-03T15:39:13-16Z** -- a ~3s automated burst that predates this
   session (the stack was down when the session began; the writes came from the v7b-armed all-final
   poller, which fired rather than being fully stopped). so there was nothing to write; the specified
   per-game precheck + finality gates correctly resolved to "already reconciled, do not write."

provenance signature of the burst: all 7 outcomes have `Source = statsapi_final` but `SourceRef = NULL`
and `Notes = NULL`, unlike the manual v7/v7b writes which carry `SourceRef = "gamePk NNNNNN"` and a
descriptive note. this is a provenance-thinness residue (see below), not a correctness defect.

## state

dai clean/synced `6c13b1d`; dai-vault at pushed v7b docs (`bd9220a`) + one pre-existing untracked file
(`06 Execution/system-state-synopsis-v1.md`, not v7c's). services brought up this session: Docker
daemon (server 29.1.3), `devcore-sql` container (SQL ready), DevCore.Api built from HEAD at Debug,
`ASPNETCORE_ENVIRONMENT=Development`, `/health` 200 on :5007. DevBypass active (tenant 1). the API has
**no** IHostedService/BackgroundService -- it cannot auto-settle; the 15:39Z burst was external.

## backlog recheck (all 7 already Final + already settled)

per-game `reconcile-precheck` (mlb_statsapi): each returns `activeRunCount 1`,
`unreconciledActiveCount 0`, `hasOutcome true` -> SingleMatch identity, but nothing unreconciled. a
`POST /reconcile` on any would 409 on the idempotency guard (outcome OR evaluation exists).

StatsAPI (`feed/live`, free) confirms all 7 Final; every score matches the settled outcome exactly:

| gamePk | AgentRunId | game | final (a-h) | side | route | lean | eval |
|--------|------------|------|-------------|------|-------|------|------|
| 823119 | 0341433e.. | LAA@SEA | 0-1 | home (SEA) | starter_enriched_market_missing | home | **correct** |
| 824335 | e440433e.. | MIA@COL | 4-14 | home (COL) | starter_missing_market_missing | null | inconclusive |
| 824416 | e540433e.. | CWS@CLE | 5-6 | home (CLE) | starter_missing_market_missing | null | inconclusive |
| 822884 | ef40433e.. | DET@TEX | 4-10 | home (TEX) | starter_missing_market_missing | null | inconclusive |
| 823935 | f640433e.. | SD@LAD | 7-12 | home (LAD) | starter_missing_market_missing | null | inconclusive |
| 824906 | ea40433e.. | STL@ATL | 11-5 | away (STL) | starter_missing_market_missing | null | inconclusive |
| 824093 | eb40433e.. | TB@KC | 5-2 | away (TB) | starter_missing_market_missing | null | inconclusive |

exactly the v7b prediction (+1 directional, +6 no-decision) -- except the directional is a **hit**, not
the "another miss" v7b expected.

## 823119 -- special attention, resolved (first correct enriched_market_missing directional)

verified at every layer, not just the denormalized field:
- artifact prose `lean`: "Slight lean toward Mariners based on starting pitching advantage."
- `artifactDirectionConsistency.structuredLeanSide = home`, `structuredLeanTeam = seattle-mariners`
- confidence 0.675, route `starter_enriched_market_missing`, regimeAllowlisted true
- StatsAPI: LAA@SEA 0-1, SEA (home) won. evaluation: `correct`, leanSide home, winningSide home.
- direction-integrity guard did not refuse (prose and structured lean agree).

**enriched_market_missing route: 0/2 -> 1/3** (823442 incorrect, 823765 incorrect, 823119 correct).
n=3 still proves nothing statistically, but the route is no longer a clean run of home-lean misses.

## pooled reassessment (rerun, full corpus)

counts: **104 reconciled / 87 directional / 17 no-decision / 10 slates.**

metrics delta (v7b -> v7c): reconciledRows(directional) 86->87 (+1, the 823119 hit);
noDecisionRows 11->17 (+6); matched 51->52; unmatched 35 (unchanged); total outcomes 97->104 (+7);
**matchRate 0.5930 -> 0.5977** (52/87).

gates: slates 10/3 MET; **enriched_market_missing n=3 MET** (was n=2); market-disagreement 4/2 MET;
confidence buckets n>=15 **NOT MET** (0.63 n=1, 0.68 n=5, 0.70 n=6, 0.72 n=2, 0.80 n=12; only 0.75
qualifies at n=61). **conclusionsAllowed: FALSE** -- unchanged gate criterion from v7/v7b.

descriptive (not actionable): overall 52/87 = **0.598**; 0.75 bucket 0.607 (n=61), **0.80 bucket 0.500**
(n=12, top band still coin-flip), 0.70 bucket 0.833 (n=6, noise); home leans **0.5714** (n=63, up from
0.5645 -- the one home hit nudged it) vs away 0.6667 (n=24, unchanged); market-agree 0.634 (n=41) vs
disagree 0.500 (n=4). route: enriched_backed_depth 0.625 (n=16), enriched_market_missing 0.333 (n=3),
unknown 0.603 (n=68).

## artifact residue analysis

- **no-decision cluster (6/6 clean).** all six new inconclusives are route `starter_missing_market_missing`
  with `leanSide null` -> the analyzer correctly abstained when both starter and market inputs were
  absent, rather than guessing. this is the abstention path working as designed; it is the healthy
  residue class, not a failure.
- **provenance-thinness residue (7/7).** the burst wrote `SourceRef = NULL` and `Notes = NULL`. the
  scores and finality are correct, but the audit trail is thinner than the manual v7/v7b writes. this
  is the one real residue defect surfaced this slice. it lives in the poller/settlement-writer path,
  **not** in prompt selection, decisioning, or the buyer artifact. deferred (see decisions); not
  backfilled this slice because that would be an unauthorised data mutation on already-settled rows.
- **directional success cluster (1/1).** 823119 is the lone directional and it is a verified hit on a
  route that previously had only misses. watch-item, not evidence.
- **failure clusters: none new.** no direction-integrity refusals, no MultipleMatches, no duplicate
  outcomes, no exclusion-reason contamination among the 7 (all `ExclusionReason NULL`, one run/gamePk).

## calibration decision

**justified: NO.** conclusionsAllowed FALSE. the binding gate (confidence buckets n>=15) is unmet on
every bucket except 0.75; the top 0.80 band is still a coin-flip at n=12; home-bias persists (0.571 vs
0.667). sample too thin and non-monotonic to tune against. continue measurement only.

## prompt-selection hardening decision

**justified: NO.** no prompt-selection defect was proven. the only defect found (provenance-thinness)
is in the settlement-writer path, is non-semantic, and does not touch route selection. per the slice
non-negotiables, prompt-selection changes are permitted only for a *proven, safely-isolated,
non-semantic provenance/test defect in that path* -- which did not occur here. no changes made; the
poller provenance gap is deferred to a future slice.

## verdict

**no tuning. verification + measurement only.** the backlog is fully settled and verified against
source of truth; the pooled read stays gate-closed on the same confidence-bucket thinness as v7/v7b.
the only forward signal is 823119 flipping enriched_market_missing off a clean-miss run (n=3), and a
provenance-thinness residue on the poller writes to clean up later.

## next action

**v8 (measurement + poller-provenance cleanup):** as new slates settle, keep re-running the pooled
report until at least one confidence bucket beyond 0.75 reaches n>=15 (the gate). separately, harden the
all-final poller so its writes carry `SourceRef`/`Notes` parity with the manual path (non-semantic,
provenance-only). do **not** tune thresholds, change prompt selection/decisioning, alter the buyer
artifact, or change the /metrics denominator while conclusionsAllowed is FALSE.

## safety ledger

paid calls 0; new game runs 0; reconciliation writes **0** (backlog already settled; precheck gated
every game to no-write); DB migrations/schema mutations 0; prompt behavior 0; decision behavior 0;
buyer-visible output 0; /metrics denominator untouched. StatsAPI reads are free (not paid). no dai code
changed -> no tests needed; code baseline unchanged at `6c13b1d`.
