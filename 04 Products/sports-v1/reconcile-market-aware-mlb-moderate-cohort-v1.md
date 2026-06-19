# Reconcile Market-Aware MLB Moderate Cohort v1

**date:** 2026-06-18 (partial) / 2026-06-19 (completion pass)
**status:** COMPLETE - 8 of 8 reconciled. All games Final; outcome/eval totals 29/29.
**classification:** reconciliation slice; docs + DB outcome/eval rows only. No code, no model, no generation.

**Anchor:** New evidence regime, new calibration baseline.

## 1. Problem Statement

Reconcile the 8 market-aware MLB Stage 0 runs (the first post-market-aware moderate cohort, captured in `market-aware-mlb-stage-0-capture-v1.md`) against actual Final game results, using the existing identity-driven reconciliation path only. Reconcile only Final/settled games; record the rest as pending. Evaluate correctness on the structured `AgentRun.LeanSide`, not artifact prose.

This cohort is a new calibration baseline. It must not be compared directly to the old pre-market thin cohort as if only the sufficiency band changed: the analyzer input itself changed (market run-line evidence is now visible to the analyzer), so the decision process is different.

## 2. Principal Engineer Review

1. **Are these still the correct 8 active market-aware MLB runs?** Yes. DB `AgentRunKey` 180013-180020 carry run-id prefixes `b0/b4/b8/bc/bd/be/c2/c4 de423e`, matching the target list exactly.
2. **Identity-bearing?** Yes. All 8 carry `SourceProvider=mlb_statsapi` and `ExternalGameId` = the captured gamePk; `ExclusionReason` is null (active) on all 8; `TenantKey=1` (dev bypass tenant).
3. **No outcome/eval rows before this slice?** Confirmed: 0 outcome / 0 eval on all 8; table totals `AgentRunOutcomes=21`, `AgentRunEvaluations=21`.
4. **Are all target games Final?** No. At `2026-06-18T13:28Z` StatsAPI reports **2 Final, 6 Preview/Scheduled** (the 6 have not started; first pitch 17:35Z onward). Per the settlement gate, only the 2 Final games were reconciled.
5. **Is the reconciliation path still correct?** Yes. `POST /api/agent-runs/reconcile` (`AgentRunsController.Reconcile`) matches on the canonical `(SourceProvider, ExternalGameId)` key, writes nothing unless `SingleMatch`, and records exactly one `AgentRunOutcome` + one derived `AgentRunEvaluation` per matched run via the shared `AddOutcomeAndEvaluation` path.
6. **Does reconcile change analyzer behavior?** No. It writes outcome/eval rows only. `LeanSide` comes from the denormalized `AgentRun` column (no `OutputJson` parsing); no prompt, model, confidence, posture, or projection mutation.
7. **bdde423e evaluation basis?** Confirmed `LeanSide=home` in the DB column; it will be evaluated on structured `home`, not prose. (It is among the 6 pending this slice, so not yet reconciled.)

Result: a clean partial reconcile gated honestly on Final settlement. Both Final games were home leans that lost to the away team, so the first 2 market-aware moderate outcomes both grade incorrect (n=2, not a calibration signal).

## 3. Cohort Definition

- 8 runs, `AgentRunKey` 180013-180020, artifact `sports_decision_artifact_v3`.
- All active (`ExclusionReason=null`), identity-bearing (`mlb_statsapi` + gamePk), `TenantKey=1`.
- All `SourceSufficiency=moderate`; all observed `PerceiveFulfillment decision=0/Fulfilled`, reason `moderate_or_rich_sufficiency`.
- All ground `starting_pitching` + `market` (-> `market_odds`).
- Structured `LeanSide`: 7 home, 1 away (`b8de423e`), 0 null.

## 4. Settlement Gate Result

Checked Final status for each gamePk via the existing StatsAPI settlement source (`statsapi.mlb.com/api/v1/schedule`) at `2026-06-18T13:28Z`.

| gamePk | run | game | start (UTC) | StatsAPI state | gate |
|---|---|---|---|---|---|
| 824992 | b0de423e | Pirates at Athletics | 2026-06-18T01:40Z | Final | reconcile |
| 823127 | b4de423e | Orioles at Mariners | 2026-06-18T01:40Z | Final | reconcile |
| 824748 | b8de423e | Blue Jays at Red Sox | 2026-06-18T17:35Z | Preview/Scheduled | pending |
| 823772 | bcde423e | Guardians at Brewers | 2026-06-18T18:10Z | Preview/Scheduled | pending |
| 822889 | bdde423e | Twins at Rangers | 2026-06-18T18:35Z | Preview/Scheduled | pending |
| 823125 | bede423e | Orioles at Mariners | 2026-06-18T20:10Z | Preview/Scheduled | pending |
| 823448 | c2de423e | Mets at Phillies | 2026-06-18T22:40Z | Preview/Scheduled | pending |
| 823533 | c4de423e | White Sox at Yankees | 2026-06-18T23:05Z | Preview/Scheduled | pending |

2 Final -> reconciled. 6 not started -> pending; no rows written for them. All 6 expected Final later today (latest first pitch 23:05Z + ~3h ≈ 2026-06-19T02:00Z).

## 5. Reconciliation Table

Reconciled runs (Final only):

| field | b0de423e | b4de423e |
|---|---|---|
| AgentRunKey | 180013 | 180014 |
| gamePk | 824992 | 823127 |
| teams (away at home) | Pittsburgh Pirates at Athletics | Baltimore Orioles at Seattle Mariners |
| scheduled start (UTC) | 2026-06-18T01:40Z | 2026-06-18T01:40Z |
| final score (home-away) | 4-12 | 3-5 |
| winner | Pirates (away) | Orioles (away) |
| settlement source | mlb_statsapi (StatsAPI schedule) | mlb_statsapi (StatsAPI schedule) |
| reconcile status | SingleMatch | SingleMatch |
| structured LeanSide | home | home |
| outcomeStatus posted | away_win | away_win |
| evaluation result | **incorrect** | **incorrect** |
| SourceSufficiency band | moderate | moderate |
| PerceiveFulfillment decision | 0 / Fulfilled (observed) | 0 / Fulfilled (observed) |
| grounded signals | starting_pitching, market | starting_pitching, market |
| market context | odds_api run line favors Athletics (home lean lost) | odds_api run line favors Mariners -1.5 (home lean lost) |
| notes | home favorite blown out 4-12 | home lean lost by 2 |

Pending runs (not reconciled this slice; no rows written):

| run | gamePk | game | start (UTC) | LeanSide | reason |
|---|---|---|---|---|---|
| b8de423e | 824748 | Blue Jays at Red Sox | 17:35Z | away | not started |
| bcde423e | 823772 | Guardians at Brewers | 18:10Z | home | not started |
| bdde423e | 822889 | Twins at Rangers | 18:35Z | home (structured) | not started |
| bede423e | 823125 | Orioles at Mariners | 20:10Z | home | not started |
| c2de423e | 823448 | Mets at Phillies | 22:40Z | home | not started |
| c4de423e | 823533 | White Sox at Yankees | 23:05Z | home | not started |

## 6. Outcome / Evaluation Summary

- Before: `AgentRunOutcomes=21`, `AgentRunEvaluations=21`.
- After: `AgentRunOutcomes=23`, `AgentRunEvaluations=23` (+2, exactly one outcome + one eval per reconciled run).
- The 6 pending runs and all other runs are unchanged; no double-write.
- DB rows verified: both reconciled runs show `LeanSide=home`, `WinningSide=away`, `EvalStatus=incorrect`, `OutcomeStatus=away_win`, `Source=mlb_statsapi`.

## 7. Correct / Incorrect / Inconclusive Totals

- This slice (2 reconciled): **0 correct / 2 incorrect / 0 inconclusive.**
- Reason: both were structured home leans; the away team won both games.
- This is n=2 within the market-aware moderate regime - directional only, not a calibration threshold.

## 8. Market-Aware Moderate-Band Confirmation

Both reconciled runs are confirmed market-aware moderate artifacts: `artifactVersion=sports_decision_artifact_v3`, `SourceSufficiency.band=moderate`, grounded critical groups `[identity_schedule, market_odds, starting_pitching]` (market evidence present), re-read live via `GET /api/agent-runs/{id}/artifact` after the outcome write.

## 9. Structured LeanSide Confirmation

Evaluation used the denormalized `AgentRun.LeanSide` column (the reconcile path never parses prose): both reconciled runs `home`. The matcher/evaluator never read artifact prose.

## 10. Note on bdde423e Structured / Prose Mismatch

`bdde423e` (gamePk 822889, Twins at Rangers) carries structured `LeanSide=home` (Rangers) while its artifact prose/factor text points toward the Minnesota Twins. DB confirms the structured column is `home`. This run did **not** reconcile this slice (game not started); when it settles it must be evaluated on structured `home`, not prose. Preserved as an artifact-quality risk for the next reconciliation pass.

## 11. No Analyzer Behavior Change

Reconciliation writes outcome/eval rows only. No prompt, model call, FastAPI invocation, confidence/posture/lean mutation, or threshold change occurred. The analyzer was not exercised; FastAPI (:8000) was never started.

## 12. Observed PerceiveFulfillment Remained Read-Only

Post-reconcile `GET /artifact` on both reconciled runs returned identical projection to capture time: `enforcementMode=observed`, `result.decision=0` (Fulfilled), `result.sufficiencyBand=moderate`, `result.reason=moderate_or_rich_sufficiency`. The outcome write did not change the read-on-demand projection; observed mode stayed neutral and non-persisted.

## 13. Recommended Next Slice

**Reconcile Market-Aware MLB Moderate Cohort v1 - settlement completion** after the remaining 6 games are Final (expected ~2026-06-19T02:00Z). Reconcile the 6 pending gamePks via the same path, evaluate on structured `LeanSide` (explicitly evaluate `bdde423e` on structured `home`), and confirm outcome/eval totals reach 29/29 if all 6 settle normally. Then a within-regime read of the full 8-run market-aware moderate cohort (correct/incorrect distribution) - compared only against other market-aware runs, never against the pre-market thin cohort.

Deferred unchanged: advisory/enforcement, buyer display, moneyline enrichment, WNBA, tenant overrides, live Probe, scheduled settlement automation.

---

## Completion Pass (2026-06-19)

The remaining 6 runs settled and were reconciled via the same `POST /api/agent-runs/reconcile`
path. No code, no model calls, no generation, no Probe, no advisory/enforcement, no buyer/frontend
changes, no migrations. Outcome/eval rows only.

### CP-1. Settlement Gate Result (all 6 Final)

Final status + scores via StatsAPI `schedule` at `2026-06-19` (home-away):

| gamePk | run | game | StatsAPI state | final (away-home) | winner |
|---|---|---|---|---|---|
| 824748 | b8de423e | Blue Jays at Red Sox | Final | 4-3 | Blue Jays (away) |
| 823772 | bcde423e | Guardians at Brewers | Final | 4-2 | Guardians (away) |
| 822889 | bdde423e | Twins at Rangers | Final | 9-3 | Twins (away) |
| 823125 | bede423e | Orioles at Mariners | Final | 0-3 | Mariners (home) |
| 823448 | c2de423e | Mets at Phillies | Final | 6-4 | Mets (away) |
| 823533 | c4de423e | White Sox at Yankees | Final | 5-1 | White Sox (away) |

All 6 Final -> all 6 reconciled.

### CP-2. Pre-reconcile gates (per run, before any write)

All 6 confirmed: active (`ExclusionReason=null`), identity-bearing (`mlb_statsapi` + gamePk),
`Status=completed`, `artifactVersion=sports_decision_artifact_v3`, `SourceSufficiency.band=moderate`,
`PerceiveFulfillment result.decision=0 (Fulfilled)` reason `moderate_or_rich_sufficiency`,
`TenantKey=1`, and 0 existing outcome / 0 existing eval rows. Table totals before: 23/23.

### CP-3. Reconciliation Table (completion pass)

| field | b8de423e | bcde423e | bdde423e | bede423e | c2de423e | c4de423e |
|---|---|---|---|---|---|---|
| AgentRunKey | 180015 | 180016 | 180017 | 180018 | 180019 | 180020 |
| gamePk | 824748 | 823772 | 822889 | 823125 | 823448 | 823533 |
| teams (away at home) | Blue Jays at Red Sox | Guardians at Brewers | Twins at Rangers | Orioles at Mariners | Mets at Phillies | White Sox at Yankees |
| final (home-away) | 3-4 | 2-4 | 3-9 | 3-0 | 4-6 | 1-5 |
| winner | away | away | away | home | away | away |
| reconcile status | SingleMatch | SingleMatch | SingleMatch | SingleMatch | SingleMatch | SingleMatch |
| structured LeanSide | away | home | **home** | home | home | home |
| outcomeStatus posted | away_win | away_win | away_win | home_win | away_win | away_win |
| evaluation result | **correct** | **incorrect** | **incorrect** | **correct** | **incorrect** | **incorrect** |
| SourceSufficiency band | moderate | moderate | moderate | moderate | moderate | moderate |
| PerceiveFulfillment | 0 / Fulfilled | 0 / Fulfilled | 0 / Fulfilled | 0 / Fulfilled | 0 / Fulfilled | 0 / Fulfilled |
| ArtifactDirectionConsistency | Consistent | Consistent | **PotentialMismatch** | Consistent | Consistent | Consistent |
| NamedRiskGrounding | DepthInsufficient | Ungrounded | Ungrounded | Ungrounded | Ungrounded | Ungrounded |

### CP-4. Outcome / Evaluation Totals

- Before completion pass: `AgentRunOutcomes=23`, `AgentRunEvaluations=23`.
- After completion pass: `AgentRunOutcomes=29`, `AgentRunEvaluations=29` (+6, one outcome + one eval per run).
- DB verified: each of the 6 shows the structured `LeanSide`, derived `WinningSide`, and `EvalStatus`
  above; `Source=mlb_statsapi`; no double-write (each had 0 rows before).

### CP-5. Completion-pass result

- This pass (6 reconciled): **2 correct / 4 incorrect / 0 inconclusive.**
  - correct: bede423e (home lean, Mariners won), b8de423e (away lean, Blue Jays won).
  - incorrect: bcde423e, bdde423e, c2de423e, c4de423e (all home leans, away team won).

### CP-6. Full 8-run market-aware moderate cohort (AgentRunKey 180013-180020)

- **2 correct / 6 incorrect / 0 inconclusive.**
- correct: bede423e, b8de423e. incorrect: b0de423e, b4de423e, bcde423e, bdde423e, c2de423e, c4de423e.
- n=8, within-regime only. Directional, not yet a calibration threshold. Compare only against other
  market-aware runs, never against the pre-market thin cohort.

### CP-7. Structured LeanSide confirmation (incl. bdde423e)

Evaluation used the denormalized `AgentRun.LeanSide` column on every run; the reconcile path never
parses prose. `bdde423e` (gamePk 822889) was evaluated on its structured `LeanSide=home` (Rangers),
not the prose that leans Minnesota. DB row confirms `LeanSide=home`, `WinningSide=away`,
`EvalStatus=incorrect`. The structured/prose mismatch (section 10) persists as an artifact-quality
risk and is unchanged by this pass.

### CP-8. ArtifactDirectionConsistency observations

`artifactDirectionConsistency` projected (read live via `GET /artifact`): **Consistent** on 5 runs
(b8de423e away, bcde423e/bede423e/c2de423e/c4de423e home) and **PotentialMismatch** on **bdde423e**
with warning "structured lean=home but lean prose points to the opposite side" (detected prose side
= away). The guard surfaced exactly the section-10 artifact-quality risk: bdde423e carries structured
`LeanSide=home` (Rangers) while its prose leans the Twins (away). The reconcile path still evaluated
on the structured `home` and graded it incorrect. Observation only this slice; no decision made --
the PotentialMismatch is a candidate input for the calibration-review slice's artifact-quality read.

### CP-9. NamedRiskGrounding observations

`namedRiskGrounding` rolled up to **Ungrounded** on 5 of 6 (bcde423e, bdde423e, bede423e, c2de423e,
c4de423e) and **DepthInsufficient** on b8de423e; each run carried exactly 1 warning. Pattern: named
risks in the prose map to grounded groups but at insufficient depth, or to groups not grounded at
all -- consistent with the intentionally thin current MLB data. Observation only; this pass creates
the labels, the next slice interprets them.

### CP-10. Recommended next slice

**Market-Aware MLB Moderate Calibration Review v1** -- interpret the 8 labels now recorded. Do the
within-regime read (2/6/0), examine the NamedRiskGrounding Ungrounded/DepthInsufficient pattern and
the null ArtifactDirectionConsistency on all v3 runs, and only then consider source-priority or
calibration decisions. No labels are created in that slice; it reads what this slice wrote.
