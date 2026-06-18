# Reconcile Market-Aware MLB Moderate Cohort v1

**date:** 2026-06-18
**status:** PARTIAL RECONCILE - 2 of 8 Final reconciled, 6 pending settlement.
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
