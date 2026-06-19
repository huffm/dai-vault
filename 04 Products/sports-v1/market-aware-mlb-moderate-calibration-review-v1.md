# Market-Aware MLB Moderate Calibration Review v1

**date:** 2026-06-19
**status:** REVIEW COMPLETE - interpretation + backlog prioritization. No labels created, no code, no model.
**classification:** read-only calibration review; docs-only. Reads the labels written by Reconcile Market-Aware MLB Moderate Cohort v1.

**Anchor:** New evidence regime, new calibration baseline. Interpreted within the market-aware regime only.

## 1. Executive Summary

The first fully reconciled market-aware MLB moderate cohort (8 runs, AgentRunKey 180013-180020)
graded **2 correct / 6 incorrect / 0 inconclusive**. The decisive finding is **not** the 6 losses;
it is that **all 8 runs carry an identical observed projection** -- `SourceSufficiency=moderate`,
`PerceiveFulfillment decision=0/Fulfilled`, confidence `0.75`, evidence richness `2`, posture
`monitor`, grounded `[identity_schedule, market_odds, starting_pitching]`. The 2 correct runs are
indistinguishable from the 6 incorrect on every projected dimension. So within this cohort the
readiness/confidence projection had **zero discriminating power** over directional correctness.

This extends, at n=8, the n=2 finding in `market-aware-mlb-miss-signal-gap-review-v1.md`: the gap is
**grounded source depth, not model awareness**. `starting_pitching` is grounded at probable-starter
*identity* only; `market` is the run line only. The individual misses are variance-dominated (close
1-2 run losses and two favorite blowups with realized in-game outliers), consistent with the
miss-signal-gap guardrail that no single missing signal would have flipped a lean.

Two artifact-contract signals fired as designed and are observations, not prediction errors:
`bdde423e` projected `ArtifactDirectionConsistency=PotentialMismatch` (structured `LeanSide=home`
vs prose leaning the Twins), and `NamedRiskGrounding` recurred as `Ungrounded` -- dominated by
`lineup_injury` (6/8 runs) -- with one `DepthInsufficient`.

**Direction set by this review:** the next factory-hardening move is a **Source Depth Contract**
that separates source *presence* from source *depth*, so "moderate" stops overstating readiness.
Starter quality/form enrichment is sequenced *after* that contract. Advisory/enforcement stays
deferred (a non-discriminating projection cannot earn authority). This is calibration direction +
backlog prioritization, not a calibrated threshold and not final model truth.

## 2. Cohort Definition

- 8 runs, `AgentRunKey` 180013-180020, artifact `sports_decision_artifact_v3`, all active
  (`ExclusionReason=null`), identity-bearing (`mlb_statsapi` + gamePk), `TenantKey=1`.
- All `SourceSufficiency=moderate`; all `PerceiveFulfillment decision=0/Fulfilled` reason
  `moderate_or_rich_sufficiency`; all grounded `starting_pitching` + `market` (-> `market_odds`).
- Structured `LeanSide`: 7 home, 1 away (`b8de423e`), 0 null.
- First post-`MLB Market Evidence Integration v1` cohort. New regime; not comparable to the
  pre-market thin cohort as if only the band changed (per stage-0 capture section 10).

## 3. Outcome Table

Scores shown away-home (StatsAPI Final). Reconciled 2026-06-19; totals 29/29 unchanged by this review.

| run | gamePk | teams (away at home) | structured LeanSide | final (away-home) | winner | eval |
|---|---|---|---|---|---|---|
| b0de423e | 824992 | Pirates at Athletics | home | 12-4 | away | incorrect |
| b4de423e | 823127 | Orioles at Mariners | home | 5-3 | away | incorrect |
| b8de423e | 824748 | Blue Jays at Red Sox | away | 4-3 | away | **correct** |
| bcde423e | 823772 | Guardians at Brewers | home | 4-2 | away | incorrect |
| bdde423e | 822889 | Twins at Rangers | home | 9-3 | away | incorrect |
| bede423e | 823125 | Orioles at Mariners | home | 0-3 | home | **correct** |
| c2de423e | 823448 | Mets at Phillies | home | 6-4 | away | incorrect |
| c4de423e | 823533 | White Sox at Yankees | home | 5-1 | away | incorrect |

Home leans went 1-6; the single away lean went 1-0. (Directional only; n=8.)

## 4. Observed Projection Table

All read live via `GET /api/agent-runs/{id}/artifact`. Defect class is interpretive (see rules, section 5/11).

| run | eval | band | PF | conf | rich | posture | ArtifactDirectionConsistency | NamedRiskGrounding (rollup / dominant ungrounded) | likely defect class |
|---|---|---|---|---|---|---|---|---|---|
| b0de423e | incorrect | moderate | 0 | 0.75 | 2 | monitor | Consistent | Ungrounded / lineup_injury (ProbeCandidate) | NormalVariance; sec: MarketOvertrust, NamedRiskUngrounded |
| b4de423e | incorrect | moderate | 0 | 0.75 | 2 | monitor | Consistent | Ungrounded / lineup_injury (ProbeCandidate) | NormalVariance; sec: NamedRiskUngrounded |
| b8de423e | correct | moderate | 0 | 0.75 | 2 | monitor | Consistent | DepthInsufficient / starting_pitching | (correct) SourceDepthInsufficient note |
| bcde423e | incorrect | moderate | 0 | 0.75 | 2 | monitor | Consistent | Ungrounded / lineup_injury (ProbeCandidate) | NormalVariance; sec: NamedRiskUngrounded |
| bdde423e | incorrect | moderate | 0 | 0.75 | 2 | monitor | **PotentialMismatch** | Ungrounded / lineup_injury (ProbeCandidate) | **ArtifactDirectionMismatch** (primary); sec: NamedRiskUngrounded |
| bede423e | correct | moderate | 0 | 0.75 | 2 | monitor | Consistent | Ungrounded / lineup_injury (ProbeCandidate) | (correct) NamedRiskUngrounded immaterial |
| c2de423e | incorrect | moderate | 0 | 0.75 | 2 | monitor | Consistent | Ungrounded / lineup_injury (ProbeCandidate) | NormalVariance; sec: NamedRiskUngrounded |
| c4de423e | incorrect | moderate | 0 | 0.75 | 2 | monitor | Consistent | Ungrounded / weather_park (ProbeCandidate) | NormalVariance; sec: MarketOvertrust, NamedRiskUngrounded |

Defect-class tally (interpretive): NormalVariance primary on 5 misses; ArtifactDirectionMismatch 1
(bdde423e); MarketOvertrust secondary on the 2 favorite-blowups (b0de423e, c4de423e);
NamedRiskUngrounded pervasive (7/8, observed-quality not miss-cause); SourceDepthInsufficient
systemic (identity-only starter depth on all 8, explicit on b8de423e); CounterCaseUnderweighted
systemic (confidence flat 0.75 -- counter-cases named but never moved confidence).

## 5. What the 2/6/0 Result Does and Does Not Prove

**Does NOT prove:**
- That the market-aware model is miscalibrated by any measurable amount. n=8 is directional, not a
  threshold (interpretation rule).
- That market run-line evidence "failed." It provided a correct directional read in several games;
  it was simply too coarse to discriminate, and it co-signed losing home leans.
- Anything about moderate-vs-thin: this cohort is a fresh regime and is not compared to the
  pre-market thin cohort.

**Does prove (within-regime, observed):**
- The readiness/confidence projection (`band`, `PF`, `confidence`, `richness`, `posture`) is
  **constant across both correct and incorrect runs** -> zero discriminating power on this cohort.
- The decision process tilts home (7/8) and rode market run-line + home-field; that combination
  underperformed here (1-6), while the single away lean hit (1-0). Tiny n; a hypothesis to watch,
  not a calibrated bias claim.
- The individual misses are consistent with variance, not a single recoverable missing signal
  (extends the n=2 miss-signal-gap finding to n=8).

## 6. PerceiveFulfillment and SourceSufficiency Interpretation

- **PerceiveFulfillment=Fulfilled had no predictive value here.** All 8 are `decision=0/Fulfilled`;
  2 right, 6 wrong. Same conclusion the prior thin-cohort calibration review reached for
  `FulfilledWithThinCoverage` -- a control signal with no variance cannot earn authority.
- **Moderate sufficiency overstated readiness** in the precise sense that it is computed from source
  *presence* (two grounded critical groups) with no notion of source *depth*. starter-identity +
  run-line clears "moderate," but neither is deep enough to be strong confirmation. Presence != depth.
- This is the structural root the projection layer cannot currently see: `DeriveBand` counts grounded
  decision-useful signals; it has no depth dimension, so a shallow read and a deep read would project
  the same band.

## 7. ArtifactDirectionConsistency Findings

- 7/8 `Consistent`; **1 `PotentialMismatch` (bdde423e)** with warning "structured lean=home but lean
  prose points to the opposite side" (detected prose side = away/Twins).
- The guard did exactly its job: it surfaced the section-10 structured/prose mismatch as a read-time
  observation. The reconcile path still evaluated on structured `home` (graded incorrect); on the
  structured side the home lean also lost by 6, so the prediction was wrong independent of the
  mismatch. **This is an evaluation-surface / artifact-contract finding, not a reconciliation bug.**
- Buyer/evaluation-surface risk: if a future buyer surface ever rendered prose direction while the
  evaluator graded structured side, bdde423e is the concrete case where the two disagree. The guard
  is the right observed seam to catch it; it stays observed-only, gating nothing.

## 8. NamedRiskGrounding Findings

- Rollups: 5 `Ungrounded`, 1 `DepthInsufficient` (b8de423e, starting_pitching), and the rest carry
  at least one ungrounded named risk; 1 warning each.
- **Dominant pattern: `lineup_injury` Ungrounded -> ProbeCandidate on 6/8 runs.** The analyzer
  repeatedly names a lineup/availability risk in prose (counter_case/watch_for/factors) that has no
  grounded lineup source. `c4de423e` instead names an ungrounded `weather_park` risk.
- `starting_pitching` and `market_odds` are `Grounded` in factors across the board (the two grounded
  signals); the ungrounded items are the *named-but-unsourced* risks.
- Reading: the model surfaces the correct uncertainty (it knows to ask about lineups) but has nothing
  grounded to weigh -- identical to the depth-not-awareness conclusion of the miss-signal-gap review.
  `lineup_injury` is the single most-named missing source across the cohort.

## 9. SourceDepth Implications

- The cohort makes "presence vs depth" the central seam: every lean rests on starter *identity* +
  run-line *direction*. There is no grounded starter quality/form, no team_form, no lineup grounding.
- Enriching a source before the band/projection can represent depth would silently inflate
  `evidence_richness`/band mechanics (the explicit caveat in miss-signal-gap section 5.1): a deeper
  starter signal would still count as "one grounded signal" and could push readiness without any
  graded depth behind it.
- Therefore the safe ordering is **contract before content**: define how depth is represented and
  inspected first (Source Depth Contract), then enrich starter data into that contract. This keeps
  enrichment from re-opening calibration in an ungoverned way.

## 10. Decisions and Deferred Decisions

Explicit reviewed decisions (this slice). These are sequencing/prioritization calls, not source
integrations, threshold changes, or enforcement activations.

1. **Advisory/enforcement: REMAIN DEFERRED.** Justified: the observed projection does not discriminate
   correct from incorrect on this cohort (and did not on the prior thin cohort). Authority requires a
   signal that explains outcome risk; this one does not yet. No internal or buyer advisory.
2. **Moderate sufficiency needs a depth qualifier.** Recommended direction: `SourceSufficiency` should
   carry a depth dimension so "moderate (shallow)" is distinguishable from "moderate (deep)". Not
   implemented here; it is the core requirement of the proposed Source Depth Contract.
3. **SourceDepth becomes the next contract seam.** Recommended next slice: **Source Depth Contract v1**
   (define presence-vs-depth representation + inspection; observed-only; no enrichment yet).
4. **Starting Pitching Quality/Form Enrichment WAITS until Source Depth Contract v1.** Enrichment
   before the depth contract would add another ungraded shallow signal and risk inflating band
   mechanics. Sequenced after the contract.
5. **Market run-line alone is too coarse for buyer-grade confirmation.** It is directional presence,
   not depth or consensus; it co-signed losing home leans. Acknowledged; no buyer claim rests on it.
6. **Moneyline / h2h enrichment REMAINS DEFERRED.** Revisit in a separate **Market Evidence Enrichment
   Review v1**; it is not the next lever (depth across sources beats more market granularity here).
7. **lineup_injury and team_form remain candidate sources, not next immediate code.** `lineup_injury`
   is the dominant ungrounded named risk (6/8) and the strongest source candidate, but it is gated
   behind the Source Depth Contract; no source integration in or after this slice.

Deferred unchanged: advisory/soft/hard enforcement, buyer display/track record, live Probe execution
(ProbeCandidate stays an observed flag only), settlement automation, MultipleMatches auto-eval,
moneyline/h2h, bullpen_availability, weather_park, WNBA, tenant overrides, calibration threshold
(entry 12 stays gated), and any confidence/posture/lean change.

## 11. Recommended Next Slice

**Source Depth Contract v1** -- define, observed-only and all-niche, how the platform represents and
inspects source *depth* distinct from source *presence* (e.g., a depth dimension on `SourceSufficiency`
and/or a `SourceDepth` projection), so "moderate" can no longer overstate readiness. No source
integration, no enrichment, no threshold, no advisory. This is the prerequisite that makes
**MLB Starting Pitching Quality/Form Enrichment v1** safe to do afterward, and it directly addresses
the zero-discrimination finding of this review.

Alternative/parallel candidates, both gated behind the depth contract:
- **MLB Starting Pitching Quality/Form Enrichment v1** (after Source Depth Contract v1).
- **Market Evidence Enrichment Review v1** (moneyline/h2h; lower priority than cross-source depth).

Interpretation guardrails honored: not compared to the pre-market thin cohort; n=8 not turned into a
threshold; market evidence not declared a categorical failure; advisory/enforcement not promoted;
presence separated from depth; prediction error separated from artifact-contract error; bdde423e
treated as a structure/prose mismatch finding, not a reconciliation bug.
