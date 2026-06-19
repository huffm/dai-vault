# MLB Starter-Depth Live Cohort Capture v1

**date:** 2026-06-19
**status:** CAPTURED - 6/6 live artifacts show SourceDepth starting_pitching=enriched; pending settlement.
**classification:** live generation/capture slice (model calls for generation only). Docs + 6 generated runs. No reconciliation, no code change.

**Anchor:** Presence is not depth. This cohort proves enriched starter depth reaches real artifacts.

## 1. Objective

Generate a small budgeted MLB cohort using the new starter-quality people-endpoint enrichment
(MLB Starter Enrichment Source-Path Fix v1) and verify that live artifacts show
`SourceDepth.starting_pitching = enriched` with the season pitching stats reaching the analyzer.

## 2. Generation Method

- Stack: `devcore-sql` (up), FastAPI agent-service (`:8000`, started this slice), .NET platform API
  (`:5007`, started this slice). Angular not started.
- Manual generation loop (not the calibration-report script): `GET /api/competitions/mlb/upcoming?days=3`
  returned 24 games; selected the first 6; `POST /api/agent-runs` per game (one gpt-4o-mini call each
  via the normal pipeline). Captured each run's projection read-only via `GET /api/agent-runs/{id}/artifact`.
- No reconciliation, no Probe, no advisory/enforcement, no buyer/frontend, no migration, no threshold/
  posture/confidence change, no code change (dai clean throughout).

## 3. Spend / Cost Summary

6 `gpt-4o-mini` analyze calls via the normal pipeline. Exact token spend not persisted (consistent
with prior cohorts); configured estimate ~$0.01-0.03/run => ~$0.06-0.18 total. All 6 completed; no
model/source failures, no early stop.

## 4. Per-Run Capture

All 6: artifact `sports_decision_artifact_v3`, identity-bearing (`mlb_statsapi` + gamePk), active,
`TenantKey=1`, `PerceiveFulfillment = 0/Fulfilled (moderate_or_rich_sufficiency)`, market grounded,
confidence 0.75, posture monitor, `LeanSide = home`, 0 outcome / 0 eval (pending).

| run | AgentRunKey | gamePk | matchup (away at home) | start (UTC) | Lean | conf | posture | band | grounded | SourceDepth starting_pitching | SourceDepth market_odds | ADC | NRG |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| f0a9433e | 190013 | 824264 | White Sox at Tigers | 2026-06-19T22:40Z | home | 0.75 | monitor | moderate | starting_pitching, market | **enriched** | shallow | Consistent | Ungrounded |
| f5a9433e | 190014 | 823534 | Reds at Yankees | 2026-06-19T23:05Z | home | 0.75 | monitor | moderate | starting_pitching, market | **enriched** | shallow | Consistent | Ungrounded |
| fba9433e | 190015 | 823853 | Giants at Marlins | 2026-06-19T23:10Z | home | 0.75 | monitor | moderate | starting_pitching, market | **enriched** | shallow | Consistent | Ungrounded |
| fda9433e | 190016 | 822966 | Nationals at Rays | 2026-06-19T23:10Z | home | 0.75 | monitor | moderate | starting_pitching, market | **enriched** | shallow | Consistent | Ungrounded |
| 01aa433e | 190017 | 824910 | Brewers at Braves | 2026-06-19T23:15Z | home | 0.75 | monitor | moderate | starting_pitching, market | **enriched** | shallow | **PotentialMismatch** | Ungrounded |
| 02aa433e | 190018 | 822886 | Padres at Rangers | 2026-06-20T00:05Z | home | 0.75 | monitor | moderate | starting_pitching, market | **enriched** | shallow | Consistent | Ungrounded |

## 5. SourceDepth Distribution

- `starting_pitching`: **enriched 6 / identity_only 0 / none 0** (observed from data on all 6).
- `market_odds`: **shallow 6** (run line only; no moneyline/consensus depth).

The fix flips the live MLB regime: pre-fix this slate would have read `identity_only`; with the
people-endpoint source path it reads `enriched` on every run.

## 6. Starter-Quality Fields Observed

`SourceDepth.starting_pitching.detail` on all 6: "season pitching stats available (era, whip, k, bb,
innings, games started)". The enriched fields reach the analyzer and visibly shape the read -- model
summaries name and compare the starters' season stats, e.g.:

- f5a9433e: "Cam Schlittler, who has an impressive ERA of 1.82. In contrast, Rhett Lowder of the Reds
  has a higher ERA of 4.60..."
- f0a9433e: "Tarik Skubal, who has a significantly lower ERA and WHIP compared to Brandon Eisert."
- 02aa433e: "Jacob deGrom presents a quality edge over Randy Vasquez..."

This is exactly the grounded depth the calibration review found missing in the identity-only cohort.

## 7. SourceSufficiency Non-Change

Band is `moderate` on all 6 with grounded `[starting_pitching, market]` -- identical breadth to the
identity-only cohort. Depth is nested inside the already-grounding `starting_pitching`; it adds no new
grounded signal and no new source group, and does not move the band. Confidence (0.75), posture
(monitor), and lean were not changed by this slice's tooling. Breadth and depth remain independent.

## 8. Artifact-Quality Projection Observations

- ArtifactDirectionConsistency: 5 Consistent, **1 PotentialMismatch** (01aa433e Brewers at Braves --
  structured lean vs prose direction; observed-only, gates nothing; flagged for the reconcile pass).
- NamedRiskGrounding: Ungrounded on all 6 (named risks map to ungrounded groups -- consistent with the
  still-thin non-starter coverage; observed-only).
- PerceiveFulfillment: observed `0/Fulfilled` on all 6; unchanged read-on-demand projection.

These are observed projections only; none gate, enforce, or change the recommendation.

## 9. New Regime Note

This is a distinct **starter-depth enriched** MLB regime (AgentRunKey 190013-190018). It must NOT be
pooled with the earlier identity-only market-aware moderate cohort (180013-180020) in calibration: the
analyzer input changed (season starter stats now visible). Now machine-readable via
`SourceDepth.starting_pitching = enriched`.

## 10. Pending Reconciliation List

All 6 are active, identity-bearing, and reconcile-eligible after StatsAPI reports Final (first pitch
22:40Z onward; expect Final ~2026-06-20T04:00Z+). None reconciled this slice; outcome/eval totals
unchanged at 29/29.

| run | AgentRunKey | gamePk | matchup | structured LeanSide |
|---|---|---|---|---|
| f0a9433e | 190013 | 824264 | White Sox at Tigers | home |
| f5a9433e | 190014 | 823534 | Reds at Yankees | home |
| fba9433e | 190015 | 823853 | Giants at Marlins | home |
| fda9433e | 190016 | 822966 | Nationals at Rays | home |
| 01aa433e | 190017 | 824910 | Brewers at Braves | home |
| 02aa433e | 190018 | 822886 | Padres at Rangers | home |

## 11. Recommended Next Slice

**Reconcile MLB Starter-Depth Enriched Cohort v1** after all 6 games are Final: reconcile the 6 exact
`(mlb_statsapi, gamePk)` keys via `POST /api/agent-runs/reconcile`, evaluate on structured `LeanSide`,
and read the result strictly within this enriched regime (never pooled with the identity-only cohort).
Then a depth-aware SourceSufficiency band review can use enriched-vs-identity_only as a real input.
