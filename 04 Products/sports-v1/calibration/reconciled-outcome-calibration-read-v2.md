# Reconciled-Outcome Calibration Read v2

**date:** 2026-06-24
**status:** READ-ONLY analysis. No code, model, prompt, generation, confidence, posture, lean, buyer, or
source-depth change. No tuning. Database read only (no writes).
**input:** all 47 settled outcomes/evaluations in devcore after the Directional-Contrast Cohort Settlement
Completion Pass (dai-vault 5b354f5). Supersedes the 35-run read v1 (does not overwrite it).
**method:** `sqlcmd` against container `devcore-sql`, db `devcore`. Per-run extraction of EvalStatus, LeanSide,
Confidence, EvidenceRichness, ArtifactVersion from `AgentRunEvaluations` joined to `AgentRuns.OutputJson`.
Regimes assigned by AgentRunKey range from the capture docs (never by a depth field), and never pooled.

**Anchor:** This read locates whether the corpus supports any real directional signal -- not whether one can be
constructed. It changes nothing. Hard rule: no confidence/prompt/generation/buyer/posture/depth change until a
settled, slate-balanced read shows a discrimination signal. v2 does NOT show one (see Decision).

## A. Overall performance (35 -> 47)

| metric | v1 (35) | v2 (47) | change |
|---|---|---|---|
| settled | 35 | 47 | +12 |
| decided | 31 | 40 | +9 |
| correct | 20 | 24 | +4 |
| incorrect | 11 | 16 | +5 |
| inconclusive | 4 | 7 | +3 |
| hit rate | 64.5% | **60.0%** | -4.5pp |

The 12 new settled runs (9 directional-contrast + 3 fresh-batch) lowered the decided hit rate. Critically, the
home-win base rate in the 40 decided games is **23/40 = 57.5%**, and the system leaned home on **33 of 40** decided
games. A naive "always lean home" baseline scores 57.5% here; the system scores 60.0%. The +2.5pp gap is well
inside binomial noise at n=40 (a 60% rate carries a ~+/-15pp 95% interval). The expanded corpus did not separate the
system from the base rate.

## B. Regime analysis (kept strictly separate)

| regime | n | decided | C | I | inc | hit | avg conf | advertised strength |
|---|---|---|---|---|---|---|---|---|
| v2 legacy MLB (120005-120010) | 6 | 3 | 2 | 1 | 3 | 66.7% | 0.658 | High/Medium mix (conf 0.5-0.75) |
| v3 pre-market thin (160004-6,170003-14) | 12 | 11 | 9 | 2 | 1 | **81.8%** | 0.719 | mostly High (0.75, rich<=1 -> cap applies only at rich<=1+High; these are High) |
| v3 identity-only moderate 180x | 8 | 8 | 2 | 6 | 0 | **25.0%** | 0.750 | all High (0.75, rich 2) |
| v3 enriched starter-depth 190x | 6 | 6 | 6 | 0 | 0 | **100.0%** | 0.750 | all High |
| directional-contrast 200x | 9 | 7 | 2 | 5 | 2 | **28.6%** | 0.693 | High (decided) / Medium (null leans) |
| fresh batch 210x | 3 | 2 | 2 | 0 | 1 | 100.0% | 0.667 | High (decided) / Medium (null) |
| NBA (120003-4,160003) | 3 | 3 | 1 | 2 | 0 | 33.3% | 0.705 |

Outperformers (190x 100%, 210x 100%, thin 81.8%) vs underperformers (180x 25%, 200x 28.6%, NBA 33%).

**Does any advantage survive sample-size scrutiny? No.** The split is explained by **which way each slate broke**,
not by depth, confidence, or grounding:

- **190x (6/6) vs 200x (2/7) are BOTH enriched starter-depth.** Same regime, opposite results. 190x was 6 home
  leans on an all-home-win slate; 200x was a balanced cohort on a road-heavy slate (5 of its 7 decided were away
  wins). The 100% vs 29% gap is slate outcome direction, not depth. **Enriched depth is NOT shown to help** -- the
  v1 enriched result did not replicate when the slate was not home-favorable.
- The thin cohort's 81.8% is the 170x runs, which leaned home on home-friendly slates. Same mechanism.
- The 180x 25% is a road-winner slate the system leaned home on 7/8 times.

No regime advantage is demonstrably about the artifact. Each regime's accuracy tracks its slate's home/away
outcome mix, because the system leans home almost always.

## C. Confidence analysis

| confidence | n | decided | hit |
|---|---|---|---|
| 0.80 | 1 | 1 | 100% (n=1, meaningless) |
| 0.75 | 34 | 34 | 61.8% |
| 0.70-0.72 | 4 | 4 | 50% |
| 0.675 | 1 | 1 | 0% |
| 0.45-0.65 | 6 | 0 | n/a (all null-lean inconclusive) |
| 0.375 | 1 | 0 | n/a |

- **Confidence is clustered:** 34 of 47 runs (72%) sit at exactly 0.75; every decided run is 0.675-0.80, and the
  sub-0.65 runs are all null-lean (inconclusive, never decided). Among decided runs confidence is effectively a
  single value.
- **0.80 vs 0.75:** cannot be compared -- only one 0.80 run exists.
- **0.75 vs 0.45:** every 0.45 run is a null-lean abstention (inconclusive by contract), so this is not a quality
  comparison; it is "took a side" vs "abstained."
- **Does confidence rank outcomes? No.** It does not vary among decided runs, so it carries no ranking
  information. The flat 0.75 went 21/34 (61.8%) -- indistinguishable from the corpus base rate. Confidence is not
  calibrated; it is a near-constant.

## D. Home vs away analysis

| lean | n | decided | C | I | hit | avg conf |
|---|---|---|---|---|---|---|
| home | 33 | 33 | 20 | 13 | 60.6% | 0.744 |
| away | 7 | 7 | 4 | 3 | 57.1% | 0.750 |
| null | 7 | 0 | 0 | 0 | n/a | 0.516 |

- **The system is effectively leaning home:** 33 of 40 decided leans (82.5%) are home.
- **Home performance reflects the base rate:** home-lean accuracy 60.6% vs the home-win base rate 57.5% in this
  corpus -- a ~3pp gap, inside noise. Home leans are not beating the base rate in any measurable way.
- **Away performance is not materially weaker corpus-wide** (57.1% vs 60.6%), but n=7 away leans is far too small
  to read, and it is bimodal by slate: directional cohort away leans 0/2, fresh batch away leans 2/2.
- Directional cohort (the only deliberately balanced sample): home 2/5 (40%), away 0/2 (0%) -- both below base
  rate on its road-heavy slate.

## E. Source depth analysis

| depth class | n | decided | C | I | hit |
|---|---|---|---|---|---|
| thin (v2 legacy + pre-market, rich 0-1) | 18 | 14 | 11 | 3 | 78.6% |
| identity-only moderate (180x, rich 2) | 8 | 8 | 2 | 6 | 25.0% |
| enriched starter-depth (190x+200x, rich 2) | 15 | 13 | 8 | 5 | 61.5% |

By richness band: richness 1 -> 11/14 (78.6%); richness 2 -> 13/26 (50.0%). The "richer" runs scored *worse*.

- **Deeper grounding does not improve outcomes here.** Thin scored highest and richness-2 scored at chance.
- **Enriched is not shown to help:** the enriched class is bimodal (190x 100%, 200x 29%) -- the two enriched
  cohorts sit on opposite slates, so the class mean is meaningless and the within-class contrast is pure slate.
- **The gains are slate-selection effects, not depth.** This is the v1 conclusion, now reinforced by the balanced
  enriched cohort (200x) failing to replicate 190x.

## F. Calibration health assessment

**STALLED.** Not improving (the one balanced test showed no edge, and overall hit fell toward the base rate); not
regressing (the system's mechanism is unchanged -- it is not getting functionally worse, it is being measured more
honestly); not stable-with-signal (no signal has appeared). The expanded corpus moved the answer from "unproven"
toward "mild evidence against a directional edge," still at insufficient sample size to be definitive.

## G. Decision

1. **Is there evidence of directional discrimination?** No. The corpus is consistent with "lean home, score the
   home base rate." The cleanest test (balanced directional cohort) returned 2/7 with away leans 0/2.
2. **Is confidence calibrated?** No. It is clustered at 0.75, does not vary among decided runs, and does not rank
   outcomes. It cannot be called calibrated.
3. **Is another balanced cohort required?** Yes. The decisive (balanced) evidence is only n=7 decided. A
   conclusion -- in either direction -- needs more balanced, mixed-slate decided games (target ~20-30 decided
   away leans).
4. **Is tuning justified?** No. There is no validated discrimination target to tune toward; tuning confidence,
   depth, or lean now would fit slate noise. (Hard rule 9; Confidence Calibration Rules v1.)
5. **Single highest-value next experiment:** a **larger Directional-Contrast Cohort Capture v2** -- deliberately
   balanced across home/away leans AND captured across several game days so it spans home-favorable and
   road-favorable slates, graded on structured LeanSide. Pair it with a **market-baseline comparison** (does the
   artifact's lean beat the closing-line favorite?), so the test is signal-vs-market, not only signal-vs-home-base-
   rate. Only that combination has the statistical power to confirm or reject directional discrimination.

## Honest conclusion (success criterion)

**Insufficient evidence to confirm signal, trending toward evidence against it.** The corpus does NOT show
predictive signal: accuracy equals the home base rate, confidence does not discriminate, and depth does not
improve outcomes. It does not yet conclusively show the *absence* of signal either, because the only slate-balanced
subset is tiny (7 decided). What the corpus says: there is no measurable directional edge so far. What it does not
say: that one is impossible -- only that none has been demonstrated, and the first clean test failed to find one.

**Recommended path: Gather more data via one targeted balanced experiment.** Not continue-as-is; not tune.

## Verification

- All figures from read-only `sqlcmd` (container `devcore-sql`, db `devcore`): 47-row settled extraction
  (EvalStatus/LeanSide/Confidence/EvidenceRichness/ArtifactVersion per run), regime assignment by AgentRunKey,
  and the home/away outcome-direction tally (home_win 23 / away_win 17 among 40 decided).
- No rows written. Totals remain AgentRunOutcomes 47 / AgentRunEvaluations 47.
- DevCore.Api was not required (read computed directly from the database).
- No code, prompt, model, confidence, posture, lean, buyer, or source-depth change.
- Cross-checked against Reconciled-Outcome Calibration Read v1: every v1 finding (home-lean skew,
  non-discriminating clustered confidence, slate-confounded regimes, enriched-not-proven) holds and is
  strengthened by the balanced 200x cohort. No contradiction with vault doctrine (dai-grill-with-vault).
