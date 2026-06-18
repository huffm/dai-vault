# Market-Aware MLB Stage 0 Capture v1

**date:** 2026-06-17
**status:** CAPTURED - pending settlement.
**classification:** docs-only capture of the first post-market-aware MLB moderate cohort.

**Anchor:** New evidence regime, new calibration baseline.

## 1. Problem Statement

MLB Market Evidence Integration v1 changed the analyzer regime: market context is now real analyzer input, not hidden metadata. This slice verifies and captures the already-generated 8-run market-aware MLB cohort so it can be reconciled later without confusing it with the pre-market thin-only cohort.

This is not a reconciliation slice. No new generation, no model call, no Probe execution, no advisory mode, no enforcement, no buyer copy, no code change, no migration, and no outcome write were performed during this pickup.

## 2. Principal Engineer Review

1. **Are the 8 generated runs recoverable from DB/artifacts?** Yes. DB rows `AgentRunKey` 180013-180020 match the exported calibration report and 8 artifact JSON files under `04 Products/sports-v1/calibration/artifacts/20260617-2009-mlb-*.json`.
2. **Are they all clearly post-market-aware regime runs?** Yes. They were generated after `MLB Market Evidence Integration v1`, all ground `market`, all include `sports_decision_artifact_v3`, and all reached `SourceSufficiency=moderate`.
3. **Do all 8 have market grounded?** Yes. `GroundedSignals=["starting_pitching","market"]` on all 8; signal availability shows `market` grounded from `odds_api`.
4. **Do all 8 have starting_pitching grounded?** Yes. Signal availability shows `starting_pitching` grounded from `mlb_statsapi` on all 8.
5. **Do all 8 project SourceSufficiency = moderate?** Yes. All 8 have critical groups `identity_schedule`, `market_odds`, and `starting_pitching` grounded.
6. **Do all 8 project PerceiveFulfillment = Fulfilled / observed?** Yes. All 8 project `decision=0` (`Fulfilled`), `enforcementMode=observed`, reason `moderate_or_rich_sufficiency`.
7. **Did any code change during this capture slice?** No. `dai` remained clean and already ahead 1 with the prior market integration commit.
8. **Did outcomes/evals remain unchanged?** Yes during pickup verification: DB stayed at `AgentRunOutcomes=21` and `AgentRunEvaluations=21`; the 8 captured runs have no outcome/eval rows.
9. **Did artifact version remain v3?** Yes. All 8 artifacts are `sports_decision_artifact_v3`.
10. **Did buyer/advisory/enforcement behavior remain unchanged?** Yes. This was artifact/DB/vault capture only.
11. **What must be captured now for future settlement/reconciliation?** Run id, active eligibility, tenant, source provider, gamePk, scheduled start, team refs, structured `LeanSide`, artifact version, source sufficiency, PerceiveFulfillment projection, grounded signals, market run-line summary, and the Final-only reconciliation gate.
12. **What would contaminate this cohort?** Regeneration, model reruns, changing LeanSide after generation, reconciling before Final, adding moneyline enrichment, changing confidence/posture rules, activating advisory/enforcement, executing Probe, buyer-copy changes, superseding/excluding without an audit note, or comparing this cohort directly against pre-market thin-only runs as if only the sufficiency label changed.

## 3. Product Architect Review

This is the first real MLB cohort where market context is part of the decision process. It gives calibration a new cell: market-aware, starter-plus-run-line, moderate sufficiency, observed Fulfilled. The cohort should later be reconciled as its own baseline, not used to claim that moderate is better than old thin runs.

No buyer-facing value proposition ships here. The useful product learning comes after settlement: whether a market-aware moderate read behaves differently from prior thin reads, and whether the run-line evidence is too coarse to support later buyer explanations without moneyline or richer market context.

## 4. Generation Method And Spend

Recovered from `04 Products/sports-v1/calibration/20260617-2009-mlb-calibration.md`:

- command path: existing `run-artifact-calibration.ps1` flow through the normal platform API.
- competition: `mlb`.
- take: `8`.
- days: `3`.
- api base url: `http://localhost:5007`.
- runs created: `8`.
- artifacts fetched: `8`.
- DB run window: `2026-06-18T00:08:36Z` through `2026-06-18T00:09:36Z`.
- exact token/cost spend: not persisted in the report, DB, or artifact export. Recoverable fact is 8 normal analyzer calls were made before this pickup; this pickup made 0 model calls.

## 5. Artifact Id Note

There is no separate persisted artifact id. The artifact endpoint and persisted `OutputJson` are keyed by `AgentRunId`. The exported artifact file name is captured below as the artifact/export identifier.

## 6. Run Identity Capture

| run id | artifact/export | tenant | competition | gamePk / external id | source provider | teams | scheduled start utc | active/eval state |
|---|---|---:|---|---|---|---|---|---|
| `b0de423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-b0de423e.json` | 1 | mlb | 824992 | mlb_statsapi | Pittsburgh Pirates at Athletics | 2026-06-18T01:40:00Z | active, no outcome/eval |
| `b4de423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-b4de423e.json` | 1 | mlb | 823127 | mlb_statsapi | Baltimore Orioles at Seattle Mariners | 2026-06-18T01:40:00Z | active, no outcome/eval |
| `b8de423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-b8de423e.json` | 1 | mlb | 824748 | mlb_statsapi | Toronto Blue Jays at Boston Red Sox | 2026-06-18T17:35:00Z | active, no outcome/eval |
| `bcde423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-bcde423e.json` | 1 | mlb | 823772 | mlb_statsapi | Cleveland Guardians at Milwaukee Brewers | 2026-06-18T18:10:00Z | active, no outcome/eval |
| `bdde423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-bdde423e.json` | 1 | mlb | 822889 | mlb_statsapi | Minnesota Twins at Texas Rangers | 2026-06-18T18:35:00Z | active, no outcome/eval |
| `bede423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-bede423e.json` | 1 | mlb | 823125 | mlb_statsapi | Baltimore Orioles at Seattle Mariners | 2026-06-18T20:10:00Z | active, no outcome/eval |
| `c2de423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-c2de423e.json` | 1 | mlb | 823448 | mlb_statsapi | New York Mets at Philadelphia Phillies | 2026-06-18T22:40:00Z | active, no outcome/eval |
| `c4de423e-f36b-1410-8164-00373db4b724` | `20260617-2009-mlb-c4de423e.json` | 1 | mlb | 823533 | mlb_statsapi | Chicago White Sox at New York Yankees | 2026-06-18T23:05:00Z | active, no outcome/eval |

## 7. Projection And Market Capture

| run id | LeanSide | confidence | posture | grounded signals | missing signals | market grounded | starter grounded | SourceSufficiency | PerceiveFulfillment | mode | artifact version | market context summary | later reconciliation eligibility |
|---|---|---:|---|---|---|---|---|---|---|---|---|---|---|
| `b0de423e` | home | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor says run line favors Athletics at +1.5 | eligible after Final |
| `b4de423e` | home | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor says run line favors Seattle Mariners at -1.5 | eligible after Final |
| `b8de423e` | away | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor says run line favors Toronto Blue Jays | eligible after Final |
| `bcde423e` | home | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor says Brewers favored by the -1.5 run line | eligible after Final |
| `bdde423e` | home | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor/prose says market confidence in Twins, while structured LeanSide is home | eligible after Final; structured side drives evaluator |
| `bede423e` | home | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor says Seattle Mariners favored by the -1.5 run line | eligible after Final |
| `c2de423e` | home | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor says run line favors Phillies by -1.5 | eligible after Final |
| `c4de423e` | home | 0.75 | monitor | starting_pitching, market | none | yes | yes | moderate | 0 / Fulfilled | observed | v3 | odds_api run-line evidence; artifact factor says run line favors Yankees by -1.5 | eligible after Final |

## 8. Distributions

SourceSufficiency:

| band | count |
|---|---:|
| thin | 0 |
| moderate | 8 |
| rich | 0 |
| insufficient | 0 |

PerceiveFulfillment:

| decision | count |
|---|---:|
| Fulfilled (`decision=0`) | 8 |
| FulfilledWithThinCoverage (`decision=1`) | 0 |
| PrimaryFulfillmentRequired | 0 |
| ProbeRequired | 0 |
| BlockedNotEvaluable | 0 |

Grounding:

| signal | grounded count |
|---|---:|
| market | 8 |
| starting_pitching | 8 |

LeanSide:

| side | count |
|---|---:|
| home | 7 |
| away | 1 |
| null | 0 |

Artifact version:

| version | count |
|---|---:|
| sports_decision_artifact_v3 | 8 |

## 9. Non-Impact Confirmation

- Outcomes/evals: DB verified at `21/21`; the 8 cohort rows have no `AgentRunOutcome` or `AgentRunEvaluation`.
- Reconciliation: none performed for this cohort; reconcile only after each game is Final.
- Model calls during pickup: none. The 8 calls were already complete before this handoff was picked up.
- Code: none changed in `dai` during capture.
- Migrations: none applied; latest applied migration remains `20260615191124_AddRunMatchEligibility`.
- Probe: not executed; `ProbeRefreshMergeAudits` count verified as `0`.
- Advisory/enforcement: inactive and unchanged.
- Buyer/frontend: unchanged.
- Confidence thresholds and posture rules: unchanged.

## 10. Calibration Regime Note

This is a post-market-aware cohort. The model saw MLB run-line evidence. Therefore this cohort is a new calibration baseline and must not be compared directly against the old pre-market thin-only cohort as if only the sufficiency label changed.

Future comparison must be within the market-aware regime, or explicitly marked as a regime-change comparison.

## 11. Runs Pending Settlement

All 8 runs are identity-bearing, active, and structurally eligible for later settlement/reconciliation:

- identity key: `(SourceProvider=mlb_statsapi, ExternalGameId=gamePk)`.
- `ExclusionReason=null`.
- structured `LeanSide` is non-null on all 8.
- no outcome/eval currently exists.
- reconcile only after StatsAPI reports Final settlement.

Special capture note: `bdde423e` has a structured `LeanSide=home`, but prose/factor text points toward the Minnesota Twins. The current evaluator contract uses structured `LeanSide`, not prose parsing. This does not block capture, but it must be preserved as an artifact-quality risk during reconciliation review.

## 12. Open Risks

- Run-line evidence is coarse. It gives a market direction but not full moneyline/h2h pricing or richer market consensus.
- Moneyline/h2h enrichment is deferred until run-line evidence proves too coarse or insufficient.
- Live Odds API MLB shape has been exercised enough for this cohort, but broader validation across doubleheaders, postponed games, bookmaker availability, and stale lines remains open.
- The calibration export script's deterministic recommendation suggested confidence calibration due to `confidence_high_for_partial_evidence` on 8/8. This slice explicitly does not change confidence thresholds.
- `bdde423e` structured/prose side mismatch should be reviewed after settlement so calibration consumers do not accidentally read prose as the evaluated side.

## 13. Recommended Next Slice

**Reconcile Market-Aware MLB Moderate Cohort v1** after all 8 games are Final.

Scope for that slice:

- Read StatsAPI Final results for the 8 exact gamePks.
- Submit `/api/agent-runs/reconcile` only for Final games.
- Confirm `SingleMatch` on each active identity key.
- Report correct/incorrect/inconclusive using structured `LeanSide`.
- Re-read artifact projection after reconciliation to confirm observed metadata remains read-only.

Alternative follow-up, only if run-line evidence proves too coarse:

**MLB Moneyline Market Enrichment v1**. Keep it separate from this cohort so the baseline stays clean.
