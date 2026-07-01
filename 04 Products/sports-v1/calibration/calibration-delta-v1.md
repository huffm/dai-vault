---
title: "Calibration Delta v1"
type: "evidence-report"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Outcome Reconciliation Follow-up v4 + Calibration Delta v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - calibration
  - directional
  - confidence
  - home-bias
  - gate-4
related:
  - "06 Execution/outcome-reconciliation-follow-up-v4.md"
  - "04 Products/sports-v1/calibration/calibration-assessment-v3.md"
  - "04 Products/sports-v1/confidence-calibration-rules-v1.md"
---

# Calibration Delta v1 -- First Directional Read on the Enriched-Market-Backed-Depth Route

## purpose

Record the first directional calibration read produced by settling the v2 backlog cohort (Outcome
Reconciliation Follow-up v4). This is a **delta on the prompt-route calibration frame** (the
`/prompt-route-calibration` metrics), NOT `Calibration Assessment v4`. It reports what the 8 freshly-settled
`starter_enriched_market_backed_depth` runs say about accuracy, confidence discrimination, and directional bias
-- and explicitly holds the no-tune line (Gate 4 not met). No prompt, threshold, model, or allowlist change.

## scope and framing (do not conflate the two frames)

- **Frame 1 -- historical corpus (`calibration-assessment-v3`, frozen):** 59 valid `sports.matchup.analysis`
  runs; DAI accuracy 57.7% (30/52 decided); home base rate 51.9%; confidence non-predictive; dominant failure
  = home bias; readiness to tune = **NO**. Unchanged by this slice.
- **Frame 2 -- prompt-route calibration (this delta):** the provenance-bearing live runs tracked by the
  `/prompt-route-calibration` metrics (263 rows). This slice added the first directional outcomes to the
  `starter_enriched_market_backed_depth` registry route from the v2 cohort.

This delta lives in Frame 2. It is small (n=8 fresh) and interim; it is not a corpus-wide verdict.

## the 8 freshly-settled directional runs (v2 enriched_market_backed_depth)

| gamePk | game | lean | confidence | result | eval |
|---|---|---|---|---|---|
| 822793 | Mets @ Blue Jays | home | 0.75 | away_win | incorrect |
| 823122 | Angels @ Mariners | home | 0.75 | home_win | correct |
| 823528 | Tigers @ Yankees | home | 0.75 | away_win | incorrect |
| 824096 | Rays @ Royals | away | 0.75 | away_win | correct |
| 824175 | Twins @ Astros | away | 0.75 | home_win | incorrect |
| 824661 | Padres @ Cubs | home | 0.75 | home_win | correct |
| 824907 | Cardinals @ Braves | home | 0.80 | away_win | incorrect |
| 824984 | Dodgers @ Athletics | away | 0.80 | away_win | correct |

**Batch accuracy: 4 / 8 = 0.500** -- at the coin-flip line and below the 51.9% home base rate.

## three signals (all consistent with the frozen v3 baseline)

### 1. confidence is still non-predictive

- confidence **0.75** (6 runs): 3 correct / 3 incorrect = **0.50**.
- confidence **0.80** (2 runs): 1 correct / 1 incorrect = **0.50**.

Higher stated confidence bought no accuracy. Route-level confirms it independently: the
`enriched_market_backed_depth` route's average confidence on **unmatched (wrong)** calls (0.760) is *higher*
than on **matched (correct)** calls (0.745). Confidence remains non-discriminating -- the `confidence-
calibration-rules-v1` "no tuning until discrimination" bar is still unmet.

### 2. home bias persists (the v3 dominant failure mode)

- **home leans (5):** 2 correct = **0.40**.
- **away leans (3):** 2 correct = **0.667**.
- **3 of the 4 misses are "leaned home, away won"** (822793, 823528, 824907); the 4th (824175) is the mirror
  "leaned away, home won". The directional error is still concentrated on over-favoring the home side.

### 3. market-tracking is not yet an independent edge

These runs are market-backed-depth (odds present). A market-vs-DAI disagreement read needs the per-run market
favorite, which is not in this delta's data pull; deferred to a market-comparison slice. Following the market is
not an edge (the standing v3 caution).

## route rollup after this slice

`starter_enriched_market_backed_depth` (registry): 15 total, **15/15 reconciled**, matched **10** / unmatched
**5**, **matchRate 0.667**. This pools the earlier 7 (6/1) with this slice's 8 (4/4). n=15 is still small and
single-cohort-weighted; treat 0.667 as provisional, not a route verdict.

## Gate posture (unchanged)

- **Gate 1 (integrity):** held -- 0 direction-integrity refusals; every persisted lean matched its artifact
  prose.
- **Gate 4 (calibration sufficiency):** **NOT met.** n=8 fresh (n=15 route total) cannot clear sufficiency, and
  the two headline signals (confidence non-predictive, home bias) reproduce the v3 finding rather than
  overturning it. **No tuning, no model swap, no buyer performance claim.**
- **Gate 5 (buyer claim):** locked.

## what would move this

Pool >=3 settled directional slates in Frame 2 (the 07-01 + 07-02 backlog will add the `enriched_market_
missing` route's first directional rows and more `starter_missing` coverage), then run a proper
`Calibration Assessment v4` across the pooled, market-backed, integrity-clean set -- with a market-disagreement
sub-read (n>1). Until then this stays a delta, not a verdict.

## paid-call status

**None.** Derived entirely from settled outcomes (Follow-up v4) and the read-only `/prompt-route-calibration`
metrics + `/rows` endpoints.
