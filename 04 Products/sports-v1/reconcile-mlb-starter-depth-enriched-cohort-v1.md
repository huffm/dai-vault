# Reconcile MLB Starter-Depth Enriched Cohort v1

**date:** 2026-06-19
**status:** WAIT-ONLY - settlement gate not met. 0 of 6 reconciled; all 6 pre-start. No rows written.
**classification:** reconciliation slice (settlement-gated). No reconciliation performed; docs-only. No code, no model, no generation.

**Anchor:** Presence is not depth. Evaluate this cohort strictly within the enriched starter-depth regime.

## 1. Problem Statement

Reconcile the 6 MLB starter-depth enriched cohort runs (AgentRunKey 190013-190018) against Final game
results via the existing identity-driven reconcile path, evaluating on structured `AgentRun.LeanSide`.
Reconcile only Final/settled games.

## 2. Settlement Gate Result

Checked Final status for each gamePk via StatsAPI `schedule` at `2026-06-19` (post-generation, pre
first-pitch). **All 6 games are Pre-Game (not started); 0 Final.** Per the settlement gate, no game was
reconciled; no outcome/evaluation row was written.

| gamePk | run | game | first pitch (UTC) | StatsAPI state | gate |
|---|---|---|---|---|---|
| 824264 | f0a9433e | White Sox at Tigers | 2026-06-19T22:40Z | Pre-Game | pending |
| 823534 | f5a9433e | Reds at Yankees | 2026-06-19T23:05Z | Pre-Game | pending |
| 823853 | fba9433e | Giants at Marlins | 2026-06-19T23:10Z | Pre-Game | pending |
| 822966 | fda9433e | Nationals at Rays | 2026-06-19T23:10Z | Pre-Game | pending |
| 824910 | 01aa433e | Brewers at Braves | 2026-06-19T23:15Z | Pre-Game | pending |
| 822886 | 02aa433e | Padres at Rangers | 2026-06-20T00:05Z | Pre-Game | pending |

Reason: this reconcile pass ran the same day the cohort was generated, before first pitch. The games
settle later tonight; expect Final ~2026-06-20T04:00Z+. This is a timing block only -- no identity,
tenant, source, or matcher issue.

## 3. Cohort Integrity (read-only, confirmed)

All 6 unchanged since capture and reconcile-eligible once Final: `Status=completed`, artifact
`sports_decision_artifact_v3`, identity-bearing (`mlb_statsapi` + gamePk), active
(`ExclusionReason=null`), `TenantKey=1`, structured `LeanSide=home`, **0 outcome / 0 eval** each.
Observed projections (from MLB Starter-Depth Live Cohort Capture v1, unchanged): SourceDepth
`starting_pitching=enriched` 6/6 and `market_odds=shallow` 6/6; SourceSufficiency band `moderate` 6/6;
PerceiveFulfillment `0/Fulfilled` 6/6; ArtifactDirectionConsistency 5 Consistent / 1 PotentialMismatch
(190017 Brewers at Braves); NamedRiskGrounding Ungrounded 6/6.

## 4. Outcome / Evaluation Totals

- Before: `AgentRunOutcomes=29`, `AgentRunEvaluations=29`.
- After: **unchanged 29/29** -- nothing was reconciled, nothing written.
- If all 6 settle normally and reconcile next pass: 35/35.

## 5. Correct / Incorrect / Inconclusive

Not applicable this pass -- 0 reconciled. The 6-run enriched cohort totals will be produced on the
settlement-completion pass.

## 6. Structured LeanSide Note (incl. 190017)

When this cohort settles it must be evaluated on the denormalized `AgentRun.LeanSide` column (all 6 =
`home`), never prose. In particular **190017 / 01aa433e (gamePk 824910, Brewers at Braves)** carries
`ArtifactDirectionConsistency=PotentialMismatch`; it must be graded on structured `home`, not its
prose direction. Preserved as an artifact-quality risk for the reconcile pass.

## 7. Non-Impact Confirmation

No reconciliation performed; no `POST /api/agent-runs/reconcile` submitted; no in-progress/0-0 score
posted; no outcome fabricated. No code, no model call (no service exercised beyond read-only DB/artifact
and the public StatsAPI settlement check), no generation, no Probe, no advisory/enforcement, no
buyer/frontend, no migration, no confidence/posture/band change. dai clean.

## 8. Recommended Next Slice

**Reconcile MLB Starter-Depth Enriched Cohort v1 -- settlement-completion pass**, after all 6 games are
Final (~2026-06-20T04:00Z+). Reconcile the 6 exact `(mlb_statsapi, gamePk)` keys via
`POST /api/agent-runs/reconcile`, evaluate on structured `LeanSide` (grade 190017 on structured `home`),
confirm totals reach 35/35 if all settle normally, and report the enriched-cohort correct/incorrect/
inconclusive totals -- read strictly within the enriched regime, never pooled with the identity-only
market-aware cohort (180013-180020). Interpretation (MLB Starter-Depth Enriched Cohort Review v1) is a
separate later slice.
