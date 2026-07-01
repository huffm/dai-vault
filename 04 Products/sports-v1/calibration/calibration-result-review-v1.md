---
title: "Calibration Result Review v1"
type: "evidence-report"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Calibration Result Review v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - calibration
  - outcome
  - provenance
  - metrics
related:
  - "04 Products/sports-v1/calibration/calibration-delta-v1.md"
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v4.md"
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v6.md"
---

# Calibration Result Review v1 -- Evidence Report

## executive summary

**The reconciled evidence does not reveal a correctable system defect that must be fixed before more paid
analyses. The correct engineering move is to gather more settled data first (Decision Option 1).** The only
provenance-bearing directional evidence is n=16 (enriched_market_backed_depth, across two slates); the
enriched_market_missing route has **zero** settled directional outcomes; and the per-confidence-bucket counts are
n<=8. Confidence is empirically non-predictive in this sample (non-monotonic: 0.70 -> 1.00, 0.75 -> 0.375,
0.80 -> 0.75), but the buckets are too small to justify a confidence or advertised-strength change. No tuning,
prompt, allowlist, model, source-strategy, or buyer change is warranted from this evidence. The highest-value
**free** next step is to settle the 9 pending 07-02 backlog games (Follow-up v7) -- especially the 3
enriched_market_missing directional runs -- before spending on any new paid cohort.

## data sources

- Read-only `GET /api/agent-runs/prompt-route-calibration/metrics` and `/rows` (tenant 1, API :5007). **Cross-check
  passed exactly:** total outcomes 95, noDecisionRows 11, directional reconciledRows 84, matched/unmatched 51/33,
  matchRate 0.6071; enriched_market_backed_depth registry route 15/15, 10/5, 0.667, confMatched 0.745 <
  confUnmatched 0.760. Endpoint data == known values (no discrepancy).
- Committed docs: `calibration-delta-v1`, `outcome-reconciliation-follow-up-v4`/`-v6`,
  `calibration-metrics-export-2026-06-30`, `okf-migration-closeout-v1`, `handoffs/current-slice.md`.
- No data written. No paid calls. No game runs. No reconciliation writes.

## sample-size warning (read this first)

- **Provenance-bearing directional outcomes = 16** (enriched_market_backed_depth: 15 registry + 1 assembly_error
  fallback), spread over **2 settled slates**. This is the only evidence that speaks to the current
  registry-authoritative system.
- **enriched_market_missing directional outcomes = 0** (3 runs pending, 07-02). No read on that route exists.
- The aggregate matchRate 0.6071 (51/33) is **dominated by 68 legacy `unknown`-route directional rows**
  (`promptSource=null`, pre-provenance), which are a historical backdrop, NOT the calibration target.
- Confidence buckets in the target sample are n=4 (0.70), n=8 (0.75), n=4 (0.80). **Nothing here clears Gate 4.**
  Every finding below is explicitly qualified by these counts.

## reconciled sample summary

95 reconciled outcomes = 84 directional + 11 no-decision. Segmentation:

| segment | rows | note |
|---|---|---|
| enriched_market_backed_depth directional (registry) | 15 | 10 correct / 5 incorrect = 0.667 |
| enriched_market_backed_depth directional (assembly_error, live fallback) | 1 | 1 incorrect (823529) |
| enriched_market_missing directional | 0 | pending (07-02) |
| starter_missing no-decision (registry, provenance) | 3 | 824338, 825066, 824818 -- all lean-null |
| legacy `unknown`-route directional (no provenance) | 68 | backdrop; matchRate ~0.594 |
| legacy `unknown`-route no-decision | 8 | lean-null |

## directional performance (enriched_market_backed_depth, n=16, two slates)

| slate | n | correct | incorrect | accuracy |
|---|---|---|---|---|
| Readiness batch (06-29 games) | 8 | 6 | 2 | 0.750 |
| v4 batch (06-30 games) | 8 | 4 | 4 | 0.500 |
| combined regime | 16 | 10 | 6 | 0.625 |
| registry-only (excl. assembly_error) | 15 | 10 | 5 | 0.667 |

The two slates disagree (0.75 vs 0.50) at n=8 each -- exactly the slate-to-slate variance you expect at this
sample size. **Do not read either slate as a regime verdict.** Home base rate ~0.52; the combined 0.625/0.667 is
inside plausible noise at n=15-16.

## confidence behavior

Per calibrated-confidence bucket (enriched_market_backed_depth, n=16):

| confidence | n | correct | accuracy |
|---|---|---|---|
| 0.70 | 4 | 4 | **1.000** |
| 0.75 | 8 | 3 | **0.375** |
| 0.80 | 4 | 3 | **0.750** |

**Non-monotonic.** The lowest-confidence bucket did best, the middle bucket worst. Route-level confirms it
independently: average confidence on unmatched (wrong) calls (0.760) exceeds that on matched (correct) calls
(0.745). **Confidence did not correlate with correctness in this sample.** Caveat: n=4/8/4 -- this is enough to
say confidence is *not yet demonstrably predictive*, NOT enough to prove it is anti-predictive.

## analyzer confidence vs calibrated confidence

- **All 16 enriched/registry rows: `analyzerConfidence == confidence`** -- no calibration shrink is applied on the
  provenance-bearing path.
- **Legacy `unknown` rows: `confidence < analyzerConfidence`** (~0.9x; e.g. 0.80 -> 0.72, 0.75 -> 0.675,
  0.70 -> 0.63) -- a shrink IS applied there.

This asymmetry is a **supported data observation**. Whether it is a defect (the calibration haircut is bypassing
the registry path) or intended (the registry path emits an already-final value) is **unproven** and cannot be
resolved from `/rows` alone. It is a candidate for a cheap read-only diagnostic later, not a change now.

## evidence richness behavior

`evidenceRichness == 2` for **all 16** enriched rows -- zero variance. It therefore **cannot** explain any miss
within the target sample (a constant discriminates nothing). No source-depth-vs-outcome signal is assessable until
a route with varying richness (e.g. enriched_market_missing, or thin regimes) settles directional outcomes.

## home / away lean behavior

| lean | n | correct | accuracy |
|---|---|---|---|
| home | 9 | 5 | 0.556 |
| away | 7 | 5 | 0.714 |

Misses (6): **4 of 6 are "leaned home, away won"** (823529, 823528, 822793, 824907); 2 are "leaned away, home
won" (823771, 824175). The direction of the signal (home leans underperform) is consistent across both slates and
with the v3 baseline, **but** (a) n=6 misses, and (b) the legacy `unknown` pool is overwhelmingly home-leaning, so
a home-bias reading risks being a base-rate artifact. **Plausible, not proven.**

## no-decision behavior

- 3 provenance starter_missing outcomes (824338 starter_missing_market_backed_depth; 825066 + 824818
  starter_missing_market_missing) all have `leanSide=null`, low confidence (0.375-0.585), and were **correctly
  recorded as no-decision** -- they stayed OUT of the directional matchRate (they are in noDecisionRows).
- All 11 no-decision rows (3 provenance + 8 legacy) are lean-null and excluded from matched/unmatched. **Behaving
  exactly as designed.**
- Low coverage on starter_missing is **appropriate caution**, not a defect: abstaining when the starter is unknown
  protects buyer trust and keeps the directional signal clean. This is working as intended.

## failure-mode hypotheses

| hypothesis | verdict | basis |
|---|---|---|
| home-lean bias | **plausible but unproven** | home 0.556 < away 0.714, 4/6 misses home-lean; n=16/6, base-rate confound |
| confidence non-predictiveness | **supported (in-sample)** | non-monotonic buckets; confU 0.760 > confM 0.745; small n limits strength |
| market-following weakness | **impossible to assess** | `/rows` carries no market favorite / de-vig field |
| evidenceRichness insufficiency | **impossible to assess** | evR constant = 2 in target sample (no variance) |
| enriched_backed_depth overconfidence | **plausible but unproven** | uniform adv=High at 0.625-0.667 accuracy; n=16 |
| starter_missing caution working as intended | **supported** | 3/3 correctly no-decision, low conf, excluded from matchRate |
| need for more source-depth features | **plausible but unproven** | cannot test without evR variance; deferred |
| need for prompt/evaluator framing change | **plausible but unproven** | leaned-home-away-won 4/6 hints at it; n=6 too small |
| calibration shrink not applied on registry path | **supported observation / unproven defect** | aConf==conf on all 16 registry rows vs shrink on legacy |

### supported findings
1. Confidence is **not yet empirically predictive** in the provenance-bearing sample (non-monotonic; confU>confM).
2. **starter_missing no-decision caution works as intended** (correct abstention, excluded from matchRate).
3. **analyzerConfidence == confidence on the registry path** (no shrink applied there), unlike legacy rows.

### plausible but unproven
Home-lean bias; enriched_backed_depth advertised-strength over-generosity; need for prompt/evaluator framing or
source-depth changes; the aConf==conf asymmetry being a defect.

### unsupported / not assessable
Market-following weakness (no market field) and evidenceRichness insufficiency (no variance) **cannot be assessed**
with the current export. No hypothesis was *refuted* -- the sample is simply too small to confirm most of them.

## engineering recommendation (exactly one primary action)

**Decision Option 1 -- No system change yet; gather more settled data.**

Rationale (engineering, not product): the only system-relevant directional sample is n=16 on a single route, with
n<=8 per confidence bucket and n=0 on enriched_market_missing. Every candidate change (confidence gate, advertised
strength, prompt/evaluator framing, source depth) would be tuning against noise. The cheapest, highest-value data
is already queued and **free**: settle the 9 pending 07-02 backlog games (Follow-up v7), which adds the first
enriched_market_missing directional read and a third enriched_backed_depth-adjacent slate at zero paid cost.

Adopt, as a **docs-only interpretation stance** (no rule/threshold change), the language that **confidence is a
model-internal coherence score, not an empirical win probability**, until discrimination is demonstrated. This
review is that stance; it changes no code.

**Minimum evidence threshold before ANY confidence/advertised-strength/prompt change:**
- >= 3 settled, integrity-clean, market-backed directional slates pooled, AND
- enriched_market_missing has >= 1 settled directional read, AND
- per-confidence-bucket n large enough to see monotonicity (target >= ~15-20 per bucket), AND
- a real DAI-vs-market disagreement sample > n=1.

(If a future change is proposed once thresholds are met, the smallest first candidate is a **read-only diagnostic**
confirming whether the calibration shrink is applied on the registry path -- proposed change: log/verify the
shrink; affected surface: calibration export/diagnostic only; benefit: resolves the aConf==conf question; risk:
none (read-only); test: unit + `/rows` cross-check; rollback: n/a; evidence threshold: none, since it is
non-mutating. This is NOT part of this slice.)

## explicit non-actions (this slice)

| action | status | why |
|---|---|---|
| model swap | **REJECTED** | no evidence of a model defect; accuracy inside noise |
| prompt change | **DEFERRED** | leaned-home-away-won is n=6; below any action threshold |
| DEFAULT_ALLOWLIST change | **REJECTED** | no promotion/demotion signal in this evidence |
| buyer claim | **REJECTED** | Gate 5 locked; no proven edge over base rate/market |
| advertised-strength change | **DEFERRED** | uniform High plausibly generous, but n=16 |
| confidence tuning | **DEFERRED** | non-predictive but n<=8/bucket; would fit noise |
| source strategy change | **DEFERRED** | evR constant -> effect unassessable |
| paid cohort capture | **DEFERRED** | settle the free 07-02 backlog first |

## deferred decisions

- Confidence / advertised-strength gate changes -> until the minimum evidence threshold above is met.
- Market-following / disagreement analysis -> until the calibration export exposes a market-favorite field.
- Source-depth feature work -> until a route with evidenceRichness variance settles directional outcomes.
- The aConf==conf registry-path question -> optional read-only diagnostic, later.

## verification

- No paid calls; no game runs; no reconciliation writes; no runtime/code/prompt/allowlist/buyer/schema change.
- `/metrics` + `/rows` used read-only; cross-check matched the known values exactly (95 / 11 / 84 / 51-33 /
  0.6071). No endpoint-vs-known discrepancy.
- New OKF front matter validated with pyyaml (9 fields; type evidence-report).
- `dai` unchanged (`b2f9771`).

## next slice recommendation

Because the review says **no change yet; gather more settled data**:
1. **Outcome Reconciliation Follow-up v7** when any 07-02 game is Final (free; adds the enriched_market_missing
   directional read -- the single most informative missing data point).
2. **Live Calibration Cohort Planning v1** (planning only, NOT paid runs): design the next paid cohort to
   maximize information -- balanced home/away leans and deliberate enriched_market_missing coverage -- so that a
   later paid capture buys discrimination rather than more of the same. Surface candidates + expected regimes +
   which calls are paid and pause for explicit approval before any spend.
