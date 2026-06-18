# Settled Miss Corpus and Artifact Consistency Review v1

**date:** 2026-06-18
**status:** REVIEW - defect taxonomy + refinement backlog. Docs-only.
**classification:** quality-control inspection of settled misses. Not calibration truth, not a source/threshold decision.

**Anchor:** Misses are inspection signals, not calibration truth unless regime, source state, and outcome labels are comparable.

## 1. Executive Summary

Seven settled runs currently evaluate `incorrect`. Across them the dominant findings are **artifact-quality and contract defects, not a single missing source**:

- **NamedRiskUngrounded is universal (7/7):** every miss named the relevant risk (starter struggling, lineup, team slump, missing sharp/public) in its `counterCase`/`watchFor`, but no grounded signal backed that risk group. The model's *awareness* was correct; its *grounded evidence* was absent.
- **SourceDepthInsufficient on `starting_pitching` (5/7 MLB):** the group is grounded by probable-starter *identity/handedness* only - no form, quality, or health. This appears in **both** pre-market thin runs and market-aware moderate runs, so market integration did not fix starter shallowness; it added a second shallow signal.
- **CounterCaseUnderweighted (5/7):** the counter-case repeatedly described the actual failure mode (e.g. "if Teng struggles early", "Twins lineup could break out of their slump") yet confidence/posture did not move - confidence is clustered at **0.675-0.75 across every regime and evidence depth**.
- **MarketOvertrust (4/7):** market-plus-home-field leans with missing independent confirmation (`sharp_public`). Present in **NBA market runs pre-dating MLB market work**, so it is a market-regime pattern, not MLB-market-aware-specific.
- **EvaluationSurfaceRisk / StructuredProseMismatch:** the earliest legacy miss (`5816433E`) has structured `LeanSide=home` but prose naming a team ("Thunder") with **no structured team identity** to map it - the same structured-vs-prose ambiguity flagged for the pending `bdde423e`. This is recurring and pre-dates market-aware, not new.

Highest factory-wide leverage is therefore a **deterministic ArtifactDirectionConsistency check** plus formalizing **SourceDepth** and **NamedRiskUngrounded** as inspectable observed contracts - all doc/seam-first, all-niche, no model. Source enrichment (starter form, team form) is the highest *sport-specific* leverage but is higher cost and stays deferred.

## 2. Corpus Definition

Source: `AgentRunEvaluations` where `EvalStatus='incorrect'`, joined to `AgentRuns` + `AgentRunOutcomes` (read-only DB inspection; no services started beyond container `sqlcmd` SELECTs). Whole-table eval today: **correct 12 / incorrect 7 / inconclusive 4.** This review covers the **7 incorrect**. The 4 inconclusive (null-lean / abstention) are out of corpus and kept separate; no `incorrect` run is superseded/excluded (all 7 active).

Artifact fields were read from each run's persisted `OutputJson` (PascalCase composer artifact). `SourceSufficiency.band` and `PerceiveFulfillment.decision` are derive-on-read and **not persisted in `OutputJson`**; band/PF below are taken from prior settled reviews for the MLB cohorts and marked where not separately re-derived this slice (no API started).

## 3. Regime Separation

| regime | runs | grounded signals | artifact ver | notes |
|---|---|---|---|---|
| pre-market / thin (MLB) | 6416433E, 3D03433E, 5403433E | starting_pitching only | v2 (6416), v3 | identity/handedness depth only; band thin |
| market-aware / moderate (MLB) | B0DE423E, B4DE423E | starting_pitching + market | v3 | band moderate; market = odds run line |
| NBA market (separate sport) | 5816433E, C1F3423E | rest_schedule + market | v2 (5816), v3 | market present, `sharp_public` missing |
| superseded / excluded | (none incorrect) | - | - | - |
| settled null / abstention | (none incorrect; null-lean -> inconclusive) | - | - | 4 inconclusive kept separate |

Regimes are not directly comparable: artifact version (v2 vs v3), sport, source state, and outcome-source (`manual` seed vs `statsapi`) all differ. Cross-regime claims below are about **recurrence of a defect pattern**, not relative hit rates.

## 4. Per-Miss Table

| run / key | sport | regime | identity | teams (home vs away) | final (H-A) | leanSide | eval | grounded | prose matches structured? | named risk ungrounded? | primary defect | pregame-actionable? | normal variance? |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 5816433E / 120003 | nba | NBA market (legacy v2) | none (null identity) | n/a (prose only) | 115-122 (away_win) | home | incorrect | rest_schedule, market | ambiguous - prose names "Thunder", no team ref | yes (sharp_public named, missing) | EvaluationSurfaceRisk + MarketOvertrust | partial (sharp/public, line move) | partial |
| 6416433E / 120005 | mlb | pre-market thin (v2) | none (null identity) | Marlins vs Braves (prose) | 12-0 (home_win) | away | incorrect | starting_pitching | yes (away=Braves) | yes (home-field edge named) | SourceDepthInsufficient + CounterCaseUnderweighted | partial (starter form) | yes (12-0 blowout) |
| C1F3423E / 160003 | nba | NBA market (v3) | odds_api event | Spurs vs Knicks | 90-94 (away_win) | home | incorrect | rest_schedule, market | yes (home=Spurs) | yes (sharp_public named, missing) | MarketOvertrust + NamedRiskUngrounded | partial (sharp/public) | partial |
| 3D03433E / 170009 | mlb | pre-market thin | statsapi 824181 | Astros vs Tigers | 3-9 (away_win) | home | incorrect | starting_pitching | yes (home=Astros) | yes ("if Teng struggles" ungrounded) | CounterCaseUnderweighted + SourceDepthInsufficient | yes (starter form) | partial |
| 5403433E / 170014 | mlb | pre-market thin | statsapi 822887 | Rangers vs Twins | 2-4 (away_win) | home | incorrect | starting_pitching | yes (home=Rangers) | yes (Twins "slump" = team_form ungrounded) | SourceDepthInsufficient + CounterCaseUnderweighted | yes (starter form, team_form) | partial |
| B0DE423E / 180013 | mlb | market-aware moderate | statsapi 824992 | Athletics vs Pirates | 4-12 (away_win) | home | incorrect | starting_pitching, market | yes (home=Athletics) | yes (Pirates-vs-RHP lineup ungrounded) | MarketOvertrust + NamedRiskUngrounded | partial (starter form, team_form) | yes (O'Hearn 6 RBI) |
| B4DE423E / 180014 | mlb | market-aware moderate | statsapi 823127 | Mariners vs Orioles | 3-5 (away_win) | home | incorrect | starting_pitching, market | yes (home=Mariners) | yes ("Kirby weakness... data lacking") | SourceDepthInsufficient + MarketOvertrust | partial (starter form) | partial (Julio exit) |

Confidence/posture across all 7: posture `monitor`; confidence 0.72, 0.70, 0.675, 0.75, 0.75, 0.75, 0.75 - a tight band invariant to regime and evidence depth.

## 5. Defect Taxonomy (tested against the corpus)

| defect | definition | hits | regimes seen |
|---|---|---|---|
| StructuredProseMismatch | structured LeanSide and prose direction conflict or are ambiguous | 1 explicit (5816433E); systemic surface risk on all | legacy; (pending bdde423e) |
| NamedRiskUngrounded | artifact names a risk with no grounded signal for that group | 7/7 | all |
| SourceDepthInsufficient | a group is grounded but too shallow for the claim (starter identity, no form/health) | 5/7 (MLB) | pre-market + market-aware |
| MarketOvertrust | lean over-depends on market/home-field without independent confirmation | 4/7 | NBA market + MLB market-aware |
| CounterCaseUnderweighted | meaningful counter-case did not move confidence/posture | 5/7 | all |
| NormalVariance | result driven by in-game/postgame-only events | 2 strong (6416 12-0, B0DE O'Hearn), partial others | all |
| MissingPregameSignal | a feasible pregame signal was absent | 7/7 (team_form, lineup, sharp_public, starter form) | all |
| EvaluationSurfaceRisk | structured eval correct, but buyer-visible artifact could confuse later review | 7/7 (prose names a team; reviewer must map to home/away) | all |

Meta-defect (not in the seed taxonomy, surfaced by inspection): **ConfidenceInvariantToDepth** - confidence does not vary with source presence/depth. Recorded as an observation only; threshold changes are out of scope.

## 6. Systems / Literature Synopsis

- **Systems thinking:** misses emerge from the *interaction* of shallow sources, correct-but-ungrounded model reasoning, flat confidence, and a prose surface that re-states teams - not a single faulty component.
- **Quality control:** defects are now categorized with rough repeatability counts and candidate inspection points (a deterministic consistency check; an observed grounded-risk check). Inspection precedes automation.
- **Lean/production:** do not automate uncertainty. The repeated failure point is *grounded depth + artifact consistency*, found by inspection before building any Probe/enforcement machinery.
- **Domain-driven design:** name the real concepts - `SourcePresence`, `SourceDepth`, `NamedRiskUngrounded`, `ArtifactDirectionConsistency`, `PregameActionableGap`, `VarianceNotSourceFixable` - so the team reasons in domain terms rather than ad-hoc.
- **Safety engineering:** keep observed warning signs (a named-but-ungrounded risk, a consistency warning) strictly separate from enforcement authority. Warnings inform; they do not gate.
- **Software architecture:** design the seam (an inspectable contract: structured lean = headline = rationale = factor side; a grounded-risk projection) before any machinery (Probe trigger, enforcement). Prefer inspectable contracts over hidden behavior.

## 7. Artifact Consistency Findings

- Structured `LeanSide` drove evaluation correctly on all 7 (evaluator never parsed prose), so **no miss was caused by a structured error**.
- The **surface** is the risk: every artifact's prose names a specific team, and a later reviewer must map that team back to home/away to sanity-check the structured side. On `5816433E` this is unverifiable - structured `home`, prose "Thunder", **no team ref persisted** (legacy null-identity run). This is the same family as the pending `bdde423e` structured/prose concern, and it pre-dates market-aware.
- **Artifact-version drift:** the corpus spans `v2` (5816433E, 6416433E) and `v3`. A consistency check must be version-aware or it will false-positive on legacy shapes.
- Recommended: a deterministic `ArtifactDirectionConsistency` check (structured lean vs headline vs rationale vs factor language vs team refs), emitted as an **observed** `quality_warnings` entry via the existing `SportsQualityChecker` surface - no model, no gating.

## 8. Source-Depth Findings

- `starting_pitching` grounded by identity/handedness only is **SourcePresence without SourceDepth**. 5/7 misses leaned partly on it; counter-cases named "if the starter struggles" and the starter then struggled (3D03433E, B-cohort) - the artifact had no grounded way to weigh that risk.
- Market integration (B-cohort) added `market` but it is *also* a shallow/confirmation-grade signal (run line, `sharp_public` still missing). Two shallow signals reached `moderate` band; depth did not increase.
- Recommendation: make **SourceDepth** a first-class, inspectable concept distinct from band/presence. Band currently counts presence; it does not express whether a present group is decision-grade.

## 9. Named-Risk Grounding Findings

- 7/7 artifacts named a risk with no grounded signal for that group. The `watchFor` text is near-templated ("Any significant lineup changes or injury updates before the game") and recurs across regimes - a stock named-but-ungrounded risk.
- This is the cleanest future **Probe-trigger seam**: *a named risk whose source group is ungrounded but feasible pregame is a Probe candidate.* Define the seam now (observed, inspectable); **defer** wiring it to any Probe execution or enforcement.

## 10. Variance vs Source-Fixable

- **VarianceNotSourceFixable (do not turn into source requirements):** O'Hearn's 6-RBI game, the 12-0 blowout, Julio's in-game exit, exact bullpen-collapse and 9th-inning shapes. These are in-game/postgame-only.
- **PregameActionableGap (source-fixable, feasible before first pitch):** starter recent form/health, team recent offensive form, known lineup/IL scratches, `sharp_public`/line-movement confirmation. These could realistically have deepened the artifact (would widen counter-case / lower confidence), but **no single one would have flipped a lean** - the leans rode market + home-field and the misses were direction-correct-risk-named, magnitude-variance-driven.

## 11. Refinement Backlog

| refinement | defect addressed | scope | change type | risk | expected leverage | recommended timing |
|---|---|---|---|---|---|---|
| Artifact Direction Consistency Guard v1 | StructuredProseMismatch, EvaluationSurfaceRisk | all niches | code (deterministic, additive observed `quality_warnings`) + doc | low (read-only, version-aware) | **high** (cheap, factory-wide, no model) | near-term |
| Named Risk Grounding Review v1 | NamedRiskUngrounded | all niches | doc first (define observed seam; map named risk -> source group) | low | high (sets the Probe seam before machinery) | near-term |
| Source Depth Contract v1 | SourceDepthInsufficient | all niches | doc/contract (SourcePresence vs SourceDepth; inspectable) | low | high (re-frames band semantics without changing them) | near-term, after the two above |
| MLB Starting Pitching Quality/Form Enrichment v1 | SourceDepthInsufficient | sports-only (MLB) | code + source (statsapi season/recent form) | medium-high (analyzer-behavior; new calibration regime again) | high (sport) | after Source Depth Contract + after the 6 pending settle |
| MLB Team Form Source Enrichment v1 | MissingPregameSignal | sports-only (MLB) | code + source | medium | medium | later |
| MLB Lineup/Injury Availability Source Review v1 | MissingPregameSignal | sports-only (MLB) | doc/research first | medium | medium | later |

Out of scope / deferred (unchanged): any source integration, Probe activation, advisory/enforcement, confidence/threshold change, buyer-copy change.

## 12. Principal Architect Review (answers)

1. **Recurring patterns?** NamedRiskUngrounded (7/7), SourceDepthInsufficient on starter (5/7), CounterCaseUnderweighted (5/7), MarketOvertrust (4/7), flat confidence 0.675-0.75, prose-team / structured ambiguity + legacy null identity.
2. **Artifact-quality vs source-coverage?** Artifact-quality: NamedRiskUngrounded, CounterCaseUnderweighted, ArtifactDirectionConsistency/EvaluationSurfaceRisk, ConfidenceInvariantToDepth. Source-coverage: SourceDepthInsufficient, MissingPregameSignal.
3. **Market-aware-regime-specific?** None uniquely. MarketOvertrust already appeared in NBA market runs; starter shallowness pre-dates market. Market integration added a second shallow signal, not depth.
4. **Pre-market too?** Yes - NamedRiskUngrounded, CounterCaseUnderweighted, SourceDepthInsufficient, MarketOvertrust, and the structured/prose surface risk all appear pre-market (incl. v2 legacy).
5. **Is `starting_pitching` too shallow when identity-only?** Yes. Presence without depth; counter-cases named the starter risk and it materialized, with nothing grounded to weigh it.
6. **Should SourceDepth be first-class?** Yes - as an inspectable contract concept distinct from presence/band. Doc/seam first.
7. **Should NamedRiskUngrounded be a future Probe trigger?** Conceptually yes (it is the natural Probe seam), but **defer activation** - define the observed seam now; do not wire Probe/enforcement.
8. **Should ArtifactDirectionConsistency be a deterministic quality check?** Yes - highest factory-wide leverage: deterministic, all-niche, version-aware, observed-only, no model.
9. **Highest factory-wide leverage?** Artifact Direction Consistency Guard + formalizing SourceDepth/NamedRiskUngrounded as observed contracts (doc/seam-level, cheap, cross-niche). Starter-form enrichment is the highest *sport* leverage but costlier and re-opens the calibration regime.
10. **Remain deferred?** All source integrations, Probe activation, advisory/enforcement, confidence/threshold changes, buyer copy - and any source-priority *decision* until the 6 pending market-aware runs settle.

## 13. Recommended Next Slice

**Artifact Direction Consistency Guard v1** - the highest-leverage, lowest-risk, all-niche refinement: a deterministic, version-aware, observed `quality_warnings` check that structured lean, headline, rationale, factor language, and team refs all point to the same side (it would also flag the legacy null-identity surface). Pair it conceptually with **Named Risk Grounding Review v1** (doc-only seam definition) so the inspection contracts exist before any Probe/enforcement machinery.

Strict reminder: the **6 pending market-aware MLB runs must settle and reconcile** (after ~2026-06-19T02:00Z) before any source-priority or calibration decision; this review is factory-hardening hypothesis work only and changes no prediction, source, threshold, buyer surface, advisory/enforcement, or calibration state.
