---
title: "Live Calibration Cohort Planning v1"
type: "plan"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Live Calibration Cohort Planning v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - calibration
  - capture
  - source-depth
  - okf
related:
  - "04 Products/sports-v1/calibration/calibration-result-review-v1.md"
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v6.md"
---

# Live Calibration Cohort Planning v1 -- Plan

## purpose

Plan (unpaid) the next calibration sample-acquisition steps so that any future paid capture buys **discrimination**,
not more of the same. This is planning only: it surfaces candidate slates, expected regimes, expected paid-call
counts, and what each cohort would clarify. **It runs no paid calls, no game generations, no reconciliation writes,
and changes no prompt/allowlist/confidence/advertised-strength/source behavior.** It operationalizes Decision
Option 1 from `calibration-result-review-v1` (no system change yet; gather more settled data).

## the constraint that dominates the plan: the free path comes first

The single most informative missing data point is the **enriched_market_missing directional read (currently n=0)**.
Three such runs are ALREADY captured and pending settlement in the 07-02 backlog (gamePks 823442, 823765, 823119 --
all lean home). **Settling them costs $0** (Outcome Reconciliation Follow-up v7, event-gated on the games going
Final). Therefore:

1. **Do not spend on a paid cohort until the free 07-02 backlog is settled and reassessed.** v7 may advance the
   evidence threshold at zero cost.
2. After v7, do a pooled reassessment across all settled enriched slates (target >= 3). Only if a *specific* gap
   remains that free settlement cannot fill does a paid cohort become justified.

## current evidence position (from calibration-result-review-v1)

- enriched_market_backed_depth directional: n=16 over 2 slates (0.75, 0.50 -> combined 0.625 / registry 0.667).
- enriched_market_missing directional: **n=0** (3 pending, free).
- Confidence non-predictive in-sample; per-bucket n<=8. Home-lean 0.556 < away-lean 0.714 (n=16). starter_missing
  no-decision caution working as intended.
- Minimum threshold before ANY confidence/strength/prompt change: >=3 pooled settled directional slates + >=1
  enriched_market_missing directional read + usable per-confidence-bucket n + a market-disagreement sample > n=1.

## candidate slate characterization (free StatsAPI schedule probe, 2026-07-01)

Read-only `schedule?date=...&hydrate=probablePitcher` (free; no paid call). Regime *starter* dimension is derivable
from announced probables; the *market* dimension (backed_depth vs missing) depends on odds depth at capture time and
is NOT knowable from the schedule.

| date | games | both probables (enriched-starter cand.) | one probable (asymmetric cand.) | none (starter_missing cand.) |
|---|---|---|---|---|
| 2026-07-03 | 13 | 2 | 6 | 5 |
| 2026-07-04 | 15 | 3 | 5 | 7 |

**Planning insight:** two days out, most games lack both announced starters; the enriched-starter pool fills in
~24h before first pitch. So a real enriched cohort must be selected the **day before**, not days ahead. "One
probable" games are asymmetric-starter candidates (fail closed to live per the assembly_error diagnostic); "none"
are starter_missing (no-decision by design).

## paid-call accounting model

- ~1 paid model analysis call per game (the `sports.matchup.analysis` run). A cohort of N games ~= N paid calls.
- Free/cheap parts (StatsAPI starter + schedule reads) are not the cost driver; the model call is.
- No cohort below is authorized. Counts are estimates to inform a later go/no-go.

## cohort options (for AFTER v7 + pooled reassessment; none authorized now)

| option | games | ~paid calls | selection criteria | expected regime mix | what it clarifies |
|---|---|---|---|---|---|
| A -- enriched_market_missing top-up | 5-6 | 5-6 | both starters announced (~24h out) + thin/early market (no deep book) | mostly enriched_market_missing | first/second directional read on the enriched_market_missing route, IF the free 07-02 rows are excluded or too few |
| B -- third enriched_backed_depth slate | 8-10 | 8-10 | both starters announced + deep de-vig market; balance home/away leans | enriched_market_backed_depth | pushes enriched_backed_depth toward >=3 pooled slates; grows per-confidence-bucket n; tests home-lean signal on balanced sides |
| C -- combined discrimination slate | 10-12 | 10-12 | both starters announced + span both market depths; deliberately balanced leans | enriched_backed_depth + enriched_market_missing | maximum discrimination per dollar: both routes + confidence buckets + a real DAI-vs-market disagreement sample |

**If (and only if) a paid cohort is justified after reassessment, prefer B or C** (>=8 games, market-backed,
balanced home/away) -- they directly attack the two things the review flagged as underpowered: per-confidence-bucket
n and the home/away lean split. Cohort A is a narrow fallback only if enriched_market_missing is still empty after
v7.

## what each cohort would and would NOT resolve

- **Would help:** pooled slate count, confidence-bucket n, enriched_market_missing existence, home/away balance,
  DAI-vs-market disagreement count.
- **Would NOT resolve on its own:** the `analyzerConfidence == confidence` registry-path question (that is a cheap
  read-only diagnostic, independent of capture); market-following weakness until the calibration export exposes a
  market-favorite field; evidenceRichness-vs-outcome until a route with richness variance settles.

## sequencing / decision flow

1. **Wait for 07-02 games to go Final -> run Outcome Reconciliation Follow-up v7** (free): settle the 9 pending
   backlog runs, especially the 3 enriched_market_missing directional (823442, 823765, 823119) and the 6
   starter_missing no-decision.
2. **Pooled reassessment** (docs-only): recompute enriched_market_backed_depth and the new enriched_market_missing
   route across all settled slates; re-check confidence monotonicity, home/away split, and market-disagreement
   count against the threshold.
3. **Go/No-go on paid capture:** only if a specific, named gap remains that free data cannot fill. If yes, bring a
   concrete candidate list (selected ~24h out) + expected regime mix + exact paid-call count, and **pause for
   explicit operator approval before any paid run.**
4. If thresholds are met and a real edge appears, that is a *separate* calibration-assessment slice -- not this one.

## explicit non-actions (this slice)

- No paid model calls. No game generations/runs. No reconciliation writes.
- No prompt changes. No DEFAULT_ALLOWLIST changes. No confidence tuning. No advertised-strength changes. No
  source-strategy changes. No DB schema / buyer copy / route changes. No model swap.
- No candidate slate is committed or captured; the tables above are illustrative planning, not an authorization.

## approval gate (hard)

Any paid capture requires a follow-up message that lists: the final candidate gamePks (selected ~24h before first
pitch), each game's expected regime, the exact expected paid-call count, and the specific evidence gap it closes --
then an explicit operator "approved" before a single paid call. Absent that, this plan produces no spend.

## verification

Docs-only. Free StatsAPI schedule reads only (no paid call, no game run, no write). New OKF front matter validated
with pyyaml (type: plan). `dai` unchanged (`b2f9771`).

## next slice

**Outcome Reconciliation Follow-up v7** when any 07-02 game is Final (free; settles the enriched_market_missing
reads). Then a **pooled calibration reassessment** (docs-only) before considering any cohort here. A paid capture
is deferred behind the approval gate above.
