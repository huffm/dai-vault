# Calibration Assessment v3 -- integrity-qualified engineering baseline

**date:** 2026-06-25
**status:** READ-ONLY assessment. The first calibration read over the integrity-qualified corpus (after Decision
Encoding Integrity v1 + Mismatch Remediation v1). No tuning, no code change, no runtime/DB mutation. This document
is the canonical engineering baseline for future tuning comparisons.
**type:** assessment slice. dai-slice-runner + dai-skill-router gate + dai-grill-with-vault +
verification-before-completion + dai-agent-handoff. Database read only (sqlcmd); zero rows written.
**supersedes (for the denominator):** Reconciled-Outcome Calibration Read v2 (47-run corpus). v3 uses the
integrity-qualified 59-run denominator and is the reference going forward.

## Executive summary

- **Verified denominator: 59 valid runs** = settled (`AgentRunOutcomes` present) AND `ExclusionReason IS NULL`.
  52 decided, 7 inconclusive (null-lean abstentions).
- **DAI accuracy 57.7%** (30/52 decided). Home-win base rate on the same decided games is **51.9%** (27/52). The
  +5.8pp edge over an always-home baseline is **inside binomial noise** at n=52 (~+/-13.6pp 95% CI).
- **Confidence is not predictive.** Decided accuracy by band is non-monotonic and inverted at the top: 0.75-0.79 =
  64.3%, **0.80+ = 20%** (1/5). 42 of 52 decided runs sit at exactly 0.75. Higher confidence does NOT correspond to
  higher observed accuracy.
- **Market comparison is only possible for 13 of 59 runs** (cohort-v2 -- the only regime with captured baselines).
  There, DAI 46.2% vs market 53.8%, with **one** DAI-vs-market disagreement, which DAI lost.
- **Dominant failure mode is home bias:** 17 of 22 incorrect predictions (77%) are "leaned home, away won."
- **Readiness to tune: NO.** No regime shows a slate-independent edge; confidence does not discriminate; the
  market-disagreement sample is n=1; market baselines cover only 22% of the corpus.

## Methodology

- Source: `sqlcmd` against container `devcore-sql`, db `devcore`. Per-run extraction of `LeanSide`, `EvalStatus`,
  `WinningSide`, `Confidence`, `EvidenceRichness` from `AgentRunEvaluations` + `AgentRuns.OutputJson`, filtered to
  the valid set. Regimes assigned by `AgentRunKey` range (per Read v2 doctrine), never by a depth field, never
  pooled across regimes.
- **Valid set filter (binding doctrine, Mismatch Remediation v1):** `ExclusionReason IS NULL AND` settled. Every
  calibrated record was verified to satisfy both. No excluded, diagnostic, superseded, or invalid run is counted.
- Market favorite = de-vigged consensus moneyline captured pregame 2026-06-24 (Market Baseline Capture v1), which
  exists for cohort-v2 only. DAI-vs-actual is computed for all valid decided runs; DAI-vs-market only where a
  baseline exists.
- Cross-checked: the EvalStatus tally (correct 30 / incorrect 22 / inconclusive 7) and the decided-winner split
  (home 27 / away 25) were re-derived directly in SQL and match the per-run computation.

## 1. Verified corpus + denominator

| metric | value |
|---|---|
| total sports runs (`RunType=sports.matchup.analysis`) | 184 |
| settled (AgentRunOutcomes = AgentRunEvaluations) | 61 |
| settled AND active (`ExclusionReason IS NULL`) -- **valid denominator** | **59** |
| settled AND excluded | 2 |

Exclusion categories (all 184 sports runs; "settled" = has an outcome):

| ExclusionReason | runs | settled |
|---|---|---|
| (active / null) | 169 | 59 |
| diagnostic | 7 | 0 |
| invalid | 4 | 2 |
| superseded | 3 | 0 |
| excluded | 1 | 0 |

The 4 `invalid` runs are the integrity exclusions: Athletics@Giants (capture failure, unsettled), 077A (unsettled),
and the two remediated settled contradictions **BDDE423E + 01AA433E**. Removing those two is what moves the
denominator from 61 to **59**, and removes a false-correct (01AA, 190x) and a false-incorrect (BDDE, 180x) from the
counts. Every one of the 59 calibrated records is settled with `ExclusionReason IS NULL` (verified).

## 2. Overall calibration

| metric | value |
|---|---|
| valid runs | 59 |
| decided (correct + incorrect) | 52 |
| inconclusive (null-lean abstentions) | 7 |
| correct | 30 |
| incorrect | 22 |
| **DAI accuracy (decided)** | **57.7%** |
| home-win base rate (decided games) | 51.9% (27/52) |
| always-home baseline edge | +5.8pp (within +/-13.6pp 95% CI at n=52) |

Market accuracy is **not computable corpus-wide** -- baselines exist for only 13 of 59 runs (sec 5). The corpus is
now near-balanced in outcomes (27 home / 25 away decided), so the historical "lean home and ride the home base rate"
mechanism no longer gets a free boost from a home-heavy slate, and DAI's edge over the base rate collapses to noise.

## 3. Regime breakdown (kept strictly separate)

| regime | n | decided | C | I | inc | DAI acc | note |
|---|---|---|---|---|---|---|---|
| v3-pre-market-thin (160/170x) | 12 | 11 | 9 | 2 | 1 | 81.8% | home leans on home-favorable slates |
| 190x-enriched-starter | 5 | 5 | 5 | 0 | 0 | 100.0% | small; 01AA (false-correct) now removed |
| v2-legacy-mlb (120005-10) | 6 | 3 | 2 | 1 | 3 | 66.7% | too small |
| cohort-v2-directional (220x) | 13 | 13 | 6 | 7 | 0 | 46.2% | the only balanced + market-backed slate |
| 200x-directional-contrast | 10 | 8 | 3 | 5 | 2 | 37.5% | balanced/road-heavy slate |
| 180x-identity-moderate | 7 | 7 | 2 | 5 | 0 | 28.6% | road-winner slate; BDDE removed |
| 210x-fresh-batch | 3 | 2 | 2 | 0 | 1 | 100.0% | too small |
| NBA (120003-4,160003) | 3 | 3 | 1 | 2 | 0 | 33.3% | too small |

**Too small for any conclusion:** 190x (5), v2-legacy (3 decided), 210x (2), NBA (3). The high performers (thin
81.8%, 190x 100%) and low performers (180x 28.6%, 200x 37.5%) are the **same split Read v2 found and attributed to
slate direction, not depth**: 190x and thin are home leans on home-favorable slates; 180x/200x are home leans on
road-heavy slates. The two deliberately balanced regimes (cohort-v2 46.2%, 200x 37.5%) -- the only fair tests --
sit at or below chance. No regime demonstrates a slate-independent edge.

## 4. Confidence assessment

Decided accuracy by confidence band:

| band | decided | correct | incorrect | accuracy |
|---|---|---|---|---|
| <0.70 | 1 | 0 | 1 | 0.0% |
| 0.70-0.74 | 4 | 2 | 2 | 50.0% |
| 0.75-0.79 | 42 | 27 | 15 | 64.3% |
| 0.80+ | 5 | 1 | 4 | **20.0%** |

Confidence value distribution (all 59): 0.75 x42, 0.80 x5, 0.70/0.72 x2 each, 0.65 x2, 0.45 x2, plus single
0.375/0.50/0.54/0.675.

**Does higher confidence currently correspond to higher observed accuracy? No.** The relationship is non-monotonic
and inverted at the top -- the highest-confidence band (0.80+) is the **worst** (20%, n=5), while the 0.75 cluster
(81% of decided runs) sits at 64.3%. Confidence is effectively a near-constant 0.75 among decided runs and carries
no ranking information. (Per boundary: this is an observation only; no confidence change is recommended.)

## 5. Market disagreement assessment

Market baselines exist for **cohort-v2 only (13 valid runs; 22% of the corpus)**. The other 46 valid runs have no
captured market favorite, so DAI-vs-market is undefined for them.

Over the 13 cohort-v2 decided runs: **DAI 6/13 (46.2%) vs market 7/13 (53.8%)** on the same games.

Games where DAI lean != market favorite (the disagreement set):

| gamePk | matchup | DAI lean | market fav | conf | regime | winner | DAI correct? | market correct? |
|---|---|---|---|---|---|---|---|---|
| 824500 | Brewers @ Reds | home (Reds) | away (Brewers) | 0.75 | cohort-v2 | away | no | yes |

- disagreement count: **1**
- DAI wins: **0** / market wins: **1**

This is the single most important number for an edge claim, and it is **n=1, lost**. DAI's directional signal on
this corpus is otherwise identical to the market favorite (12 of 13 cohort-v2 leans agree with it), so DAI is
largely tracking the market and has not demonstrated any independent directional edge.

## 6. Failure taxonomy (observed, not speculative)

22 incorrect predictions:

| category | count | share |
|---|---|---|
| home bias (leaned home, away won) | 17 | 77% |
| away/underdog miss (leaned away, home won) | 5 | 23% |
| evidence insufficiency / conflicting evidence | n/a | not separable from the data (uniform richness within regimes; no per-miss signal contrast recorded) |
| null decision | (counted separately) | 7 inconclusive abstentions, not failures |
| integrity exclusion | (removed pre-count) | 2 contradictory runs excluded (BDDE, 01AA) |

**Home bias is the dominant and only well-supported failure category.** 39 of 52 decided leans (75%) are home;
home-lean accuracy is 56.4% vs away-lean 61.5% (n=13). The misses concentrate where the system leaned home and the
road team won. The data does not support finer categories (underdog overreach, evidence insufficiency) as distinct
mechanisms -- richness/sufficiency are uniform within each regime, so no per-miss evidence signal can be attributed.

## 7. Calibration readiness

**Conclusion: NO -- there is not sufficient evidence to justify tuning.**

Support:

- **Sample size:** 52 decided overall, but the only balanced AND market-backed subset is cohort-v2 (13 decided).
  The decisive DAI-vs-market disagreement sample is **n=1**. Market baselines cover only 22% of the corpus.
- **Confidence calibration:** absent. Confidence is a near-constant 0.75 and is inverted at the top band (0.80+ =
  20%). There is no calibrated confidence target to tune toward.
- **Disagreement analysis:** DAI agrees with the market on 12 of 13 cohort-v2 games and lost the one disagreement.
  No independent edge is demonstrated; tuning now would optimize against market-tracking noise.
- **Evidence regimes:** every apparent regime advantage (thin, 190x) is explained by slate direction, not depth or
  grounding; the balanced regimes are at/below chance. No regime gives a slate-independent signal to tune toward.
- **Overall:** 57.7% vs a 51.9% base rate is within noise at n=52.

Tuning before a validated discrimination target exists would fit slate noise. The corpus is now **integrity-clean**
(a prerequisite that did not hold before this slice) but **not yet statistically sufficient**.

## 8. Open questions

1. Can market baselines be back-filled for the pre-cohort-v2 regimes (180x/190x/200x), or only captured going
   forward? Without them, market comparison stays at n=13.
2. Will the analyzer lean-agreement hardening (generation side) reduce the home-bias miss rate, or is the home lean
   a separate prompt/decision artifact? (Out of scope here; flagged.)
3. How many balanced, market-backed slates are needed for an adequately-powered DAI-vs-market test? Read v2's
   target (~20-30 decided away leans across multiple slates) remains unmet (13 away leans total in the valid set).
4. Is the 0.80+ inversion (20%, n=5) noise or a real anti-signal? Needs more 0.80 runs to tell.

## 9. Verification

- Denominator query + exclusion filter run and shown (sec 1); valid set = 59, every record settled AND
  `ExclusionReason IS NULL`.
- Headline counts cross-checked in SQL independently (correct 30 / incorrect 22 / inconclusive 7; winners home 27 /
  away 25) -- match the per-run computation.
- `dai` unchanged (no code); `dai-vault` adds this doc only. `git diff --check` clean. **No runtime behavior
  changed, no tuning performed, no code modified, no DB row written.**

## 10. Recommended next slices (assessment only; no tuning)

1. **Directional-Contrast Cohort Capture v3 + Market Baseline v2:** the binding need -- more balanced,
   market-backed slates to grow the n=1 disagreement sample.
2. **Analyzer Lean-Agreement Hardening v1** (generation side, separate lane): reduce contradictory `lean_side` at
   the source; may also bear on the home-bias miss rate.
3. **Calibration Assessment v4:** re-run this exact baseline after >=2 more market-backed slates settle, using the
   `ExclusionReason IS NULL` denominator, regimes separate.
