---
title: "Backed-Depth Divergence Capture Plan v1"
type: "plan"
date: "2026-07-05"
status: "NO_SPEND_PREP -- plan only; approval-gated, not executed"
project: "DAI"
slice: "Backed-Depth Cleanup + Divergence Capture Prep -- Phase 3"
repos:
  dai: "unchanged"
  dai-vault: "docs-only (this plan)"
tags:
  - calibration
  - cohort
  - registry
  - backed-depth
  - divergence
  - plan
related:
  - "06 Execution/reports/reconciliation-last-cohort-2026-07-05-v1.md"
  - "06 Execution/reports/backed-depth-dangling-row-cleanup-2026-07-05-v1.md"
  - "06 Execution/plans/v8-targeted-measurement-cohort-plan-v1.md"
  - "06 Execution/reconciliations/source-readiness-preflight-gate-v1.md"
---

# Backed-Depth Divergence Capture Plan v1

**mode:** NO_SPEND_PREP (default). This is a plan only. No agent-service start, no sports analysis, no new agent
runs, no paid model calls occurred. PAID_CAPTURE requires explicit `PAID_CAPTURE_APPROVED=true` in the operator
instruction (absent here).

## why (measurement gap this closes)

The v8 backed-depth cohort (7 runs, settled 2026-07-05) validated reconciliation mechanics but produced **zero
DAI-market divergence**: `marketConsensusSide == leanSide` on all 7, so DAI's 5/7 is indistinguishable from
following the market favorite. `starter_enriched_market_backed_depth` is a market-informed regime, so the model is
often mechanically pulled toward the market favorite. To measure whether DAI has **any signal independent of the
market**, the next cohort must raise the probability of divergence without cherry-picking outcomes or tuning.

This is a measurement-integrity objective. Success is NOT a high hit rate.

## 1. purpose

Capture a new `starter_enriched_market_backed_depth` registry-routed MLB cohort selected to maximize the chance of
DAI-market disagreement, so a later reconciliation can honestly evaluate BOTH the DAI lean and the market-favorite
baseline on the same games.

## 2. inclusion criteria (hard, identity/eligibility)

A candidate game must have all of:
- supported sport/provider: MLB, provider `mlb_statsapi`.
- reliable provider identity: real gamePk; `externalGameId=gamePk`; home/away resolvable to team refs.
- upcoming / not-yet-started at capture time (pre-first-pitch).
- available market snapshot with sufficient book coverage (>= 5 books, matching backed_depth depth).
- a measurable market favorite (non-degenerate implied-probability split).
- source depth for the regime: enriched starter present (both probable pitchers non-TBD) AND backed_depth market.
- verified via `GET /api/agent-runs/source-readiness` == eligible for `starter_enriched_market_backed_depth`
  (reuses the exact generation retrieval; no model call, no write).
- no duplicate provider identity (one active run per gamePk; run reconcile-precheck to confirm IdentitySafe).
- no ambiguous team mapping.

## 3. divergence-seeking prefilter (soft, ranked)

Among eligible games, prefer those where DAI is least mechanically pulled to the favorite:
- close market favorite (median favorite implied prob roughly 0.50-0.58).
- modest implied-probability gap between sides (e.g. |homeIP - awayIP| <= ~0.10).
- mixed book signals / cross-book disagreement on the favorite.
- observed market movement without overwhelming consensus.
- a starter/injury/context signal plausibly material enough to move the model off the favorite.
- de-prioritize blowout favorites (implied prob >= ~0.62) -- those drove the 7/7 agreement last cohort.

The prefilter selects the *candidate slate before outcomes are known*. It never uses results.

## 4. integrity constraints

- Define and RECORD the candidate slate before any generation.
- Record EVERY generated run (agreements AND disagreements), not only divergences.
- Preserve DAI-market agreements and disagreements; do not discard misses.
- Do not tune prompts, thresholds, confidence, routing, or advertised strength after seeing results.
- Do not re-call the model for the same game to hunt a different lean.
- Do not label the cohort edge-positive unless `conclusionsAllowed` becomes TRUE under the existing calibration
  gate (>= 3 slates AND enriched_market_missing >= 1 AND bucket n >= 15 AND market-disagreement >= 2).
- Registry routing stays DEFAULT-OFF; enable only process-scoped per run (never write `.env`), revert after.

## 5. proposed paid execution limits (defaults; PAID_CAPTURE only)

- max paid runs: **12**
- model: existing configured analyzer only (`gpt-4o-mini`, `sports_analyzer.py`); no model swap.
- max model calls per run: existing architecture limit (currently **1**).
- max total cost cap: **$0.05** unless operator overrides (each run also burns 1 the-odds-api PaidExternal call).
- stop immediately on any unexpected model path, prompt-route fallback, promptSource != registry, or cost-log change.

## 6. capture outputs (record per generated run)

run ID; game identity (gamePk); matchup; sourceProvider; externalGameId; commence/start time; DAI leanSide;
confidence; evidenceRichness / source sufficiency; market favorite (marketConsensusSide); market implied
probabilities (home/away medians) if available; DAI-market agreement flag (marketAgreement); promptSource /
registry route (recipeId@version, assembledHash); selectedDataRegime / observedDataRegime / source depth
(marketBookCount); cost-log reference if available. All available read-only on
`GET /api/agent-runs/prompt-route-calibration/rows`.

## 7. success condition

A successful divergence capture means:
- the candidate slate was captured cleanly and recorded before generation;
- identities are settlement-safe (IdentitySafe, gamePk-anchored);
- the market baseline (favorite + implied probs) was recorded for every run;
- there exists at least some DAI-market disagreement, OR the report states plainly that the system still largely
  tracks the market;
- a future reconciliation can evaluate DAI and the market baseline honestly on the same games.

It is explicitly NOT "DAI got a high hit rate."

## execution status

NOT executed. To execute, re-issue with `PAID_CAPTURE_APPROVED=true` plus (a) a concrete game date, (b) named
candidate gamePks that pass Sections 2-3, and (c) confirmation of the 12-run / $0.05 caps. Capture and settlement
stay separate slices: do not reconcile the new cohort until its games are official-Final and identity-safe.
