# Reconciled-Outcome Calibration Read v1

**date:** 2026-06-22
**status:** READ-ONLY analysis. No code, model, prompt, generation, confidence, posture, lean, buyer, or
source-depth change. No tuning. Database read only (no writes).
**input:** all 35 settled outcomes/evaluations in devcore after the enriched-cohort settlement pass
(dai-vault 80679be).

**Anchor:** This read locates where misses and weak artifacts concentrate. It changes nothing. The hard
rule stands: no confidence/prompt/generation/buyer/posture/source-depth change until this read shows the
binding constraint -- and it does (see Findings 1-3).

## 1. Corpus

35 settled runs, all MLB except 3 NBA. Decided = correct + incorrect (inconclusive excluded from rates
per Outcome Reconciliation Contract v1).

- correct 20 / incorrect 11 / inconclusive 4
- decided 31 -> overall hit rate **20/31 = 64.5%**
- artifact versions: v3 x27, v2 (legacy) x8

Regimes kept separate (never pooled):

- **v2 legacy MLB** (120005-120010): evidenceRichness 1.
- **v3 pre-market thin MLB** (160004-160006, 170003-170014): evidenceRichness 0-1.
- **v3 identity-only market-aware moderate MLB** (180013-180020): evidenceRichness 2, starter identity
  only (no depth field computed).
- **v3 enriched starter-depth MLB** (190013-190018): evidenceRichness 2, `SourceDepth.starting_pitching=enriched`.
- **NBA** (120003-120004 v2, 160003 v3): evidenceRichness 2.

Note: only the 6 enriched runs carry a populated `SourceDepth` array; the feature postdates the older
cohorts, so their absent depth means "not computed," not "identity-only." Regime labels come from the
capture docs, not from a depth field on the older runs.

## 2. Per-Regime Accuracy

- v2 legacy MLB: correct 2 / incorrect 1 / inconclusive 3 -> decided 3, **2/3**.
- v3 pre-market thin MLB: correct 9 / incorrect 2 / inconclusive 1 -> decided 11, **9/11 (82%)**.
- v3 identity-only market-aware MLB: correct 2 / incorrect 6 / inconclusive 0 -> decided 8, **2/8 (25%)**.
- v3 enriched starter-depth MLB: correct 6 / incorrect 0 -> **6/6 (100%)**.
- NBA: correct 1 / incorrect 2 -> decided 3, **1/3**.

## 3. Findings

### Finding 1 -- The system almost always leans home; measured accuracy tracks the home-win base rate, not signal quality.

- Lean direction among 31 decided runs: **home 28, away 3, null 4 (inconclusive).** ~90% home.
- Home teams won **22 of 35** games (63%). The overall 64.5% hit rate is statistically indistinguishable
  from "always lean home" against a 63% home-win slate.
- Home-lean record: 18 correct / 10 incorrect (64%). Away-lean record: 2/3 (n too small to read).
- **Implication:** the artifact's directional value is currently unproven. A naive "lean home always"
  baseline would score about the same on this corpus. Discrimination, not accuracy, is the missing axis.

### Finding 2 -- Confidence and evidence richness do not discriminate hits from misses.

- Confidence is **flat at 0.75** across nearly every v3 run -- the 6/6 enriched cohort and the 2/8
  identity-only cohort carry the *same* 0.75. Legacy v2 sits at 0.70-0.72. Only one run (160004, a
  `wait`/evidenceRichness-0 run) differs at 0.375, and it is inconclusive.
- evidenceRichness is inverted by slate confounding: evidenceRichness 1 went **11/14 (79%)** decided,
  evidenceRichness 2 went **8/16 (50%)** decided -- the "richer" runs scored *worse*, because
  evidenceRichness 2 contains the road-winner 180x slate and the NBA misses.
- **Implication:** neither the raw confidence number nor the grounded-signal count carries calibration
  signal today. The buyer band gate is presenting a number that does not separate outcomes.

### Finding 3 -- Misses concentrate in the identity-only market-aware cohort (180013-180020), and that cohort is confounded with its slate.

- 180x = 2/8. Of its 8 games, 7 were `away_win`; the cohort leaned home 7 times and away once. Home
  leans went 1/7 there; the single away lean (180015) was correct.
- The enriched 190x cohort = 6/6, but all 6 were home leans on an all-`home_win` slate.
- **180x vs 190x is not a clean depth comparison:** different slates, opposite home/away win
  distributions. The 100% vs 25% gap is explained by slate outcome direction, not demonstrably by
  starter depth. **Enriched starter depth is not yet shown to improve decision quality.** (Carries
  forward the settlement-pass interpretation.)

### Finding 4 -- Weak-artifact signature (cross-referenced to the settled-miss QC review).

The prior whole-table QC review (current-slice.md) found on the settled misses: NamedRiskUngrounded
across the board, SourceDepthInsufficient (identity/handedness only) on most, CounterCaseUnderweighted
(counter-cases named the actual failure mode while confidence stayed flat), and MarketOvertrust including
NBA. This read corroborates the mechanism: artifacts named the right risks, did not down-weight
confidence for them, and leaned home anyway.

## 4. Where Misses and Weak Artifacts Concentrate (answer)

1. **Directionally:** home leans on slates where road teams win. The system's home-lean default is the
   single largest exposure.
2. **By cohort:** the identity-only market-aware moderate cohort (180013-180020), 2/8.
3. **By artifact quality:** ungrounded named risks + flat 0.75 confidence that ignores its own
   counter-case. Confidence does not move when the artifact itself names the fragility.

## 5. What This Does NOT Yet Support

- No confidence threshold change: confidence is non-discriminating, so there is no validated target to
  tune toward. Tuning now would fit noise.
- No claim that enriched depth helps or hurts: the enriched cohort has zero directional/outcome contrast.
- No prompt/posture/source-depth change: the binding constraint is data composition (home-lean skew,
  single-direction slates), not a code defect.

## 6. Recommended Next Slice

**Directional-Contrast Cohort Capture v1** (generation + reconcile, not tuning). Capture and later
reconcile an MLB cohort deliberately balanced across (a) home and away leans and (b) slates with mixed
home/away winners, at enriched starter depth, so enriched vs identity-only can be compared without slate
confounding and confidence can be tested for discrimination. Only after a discrimination signal exists
should any confidence-calibration slice run. Keep regimes separate; never pool. The reconciliation
runtime proven in the settlement pass already supports this end-to-end.

A lighter parallel option: a read-only **home-lean base-rate note** that states the naive-baseline
comparison explicitly in the buyer-internal calibration record, so the home-lean exposure is tracked as
a known limitation before any directional claim is made.

## 7. Verification

- All figures from read-only `sqlcmd` against container `devcore-sql`, db `devcore`: whole-corpus eval
  tally (20/11/4), lean x eval cross-tab, outcome-direction distribution (home_win 22 / away_win 13),
  and per-run JSON extraction of Confidence / EvidenceRichness / Posture / ArtifactVersion / SourceDepth.
- No rows written. Totals remain AgentRunOutcomes 35 / AgentRunEvaluations 35.
- No services changed beyond being up (devcore-sql + DevCore.Api :5007 from the settlement pass).
- No code, prompt, model, confidence, posture, lean, buyer, or source-depth change.
