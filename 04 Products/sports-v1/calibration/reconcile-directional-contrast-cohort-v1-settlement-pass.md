# Directional-Contrast Cohort Reconciliation v1

**status:** WAIT-ONLY (2 gate checks) -- settlement gate still not met. 0 of 10 reconciled; all 10
Preview/Scheduled (not started). No outcome/eval rows written. Totals unchanged 35/35.
**classification:** reconciliation slice (settlement-gated). No reconciliation performed; docs-only. No
code, model, generation, confidence, posture, lean, buyer, source-depth, or reconciliation-logic change.

**Anchor:** Grade only Final games, only on structured LeanSide. Presence of a captured cohort is not a
settled outcome.

## 0. Gate Re-check Log (settlement-completion attempt)

- **Check 1 -- 2026-06-22 ~11:30 EDT (capture day):** all 10 games Preview/Scheduled, pre first pitch.
  Wait-only; nothing written.
- **Check 2 -- 2026-06-22 11:59 EDT (15:59Z), settlement-completion rerun:** StatsAPI re-queried for all
  10 gamePks; still all `Preview/Scheduled`, `totalGamesInProgress=0`. First pitches are 18:10-19:45Z
  *today* (still ~2-4h away); games are not expected Final until ~2026-06-23T04:00Z+. The "games should
  be settled" premise did not yet hold at re-check time. Gate not met again -> no reconciliation, no rows
  written, totals remain 35/35. The cohort is unchanged and reconcile-eligible the moment the games go
  Final; rerun this slice after ~2026-06-23T04:00Z.

## 1. Summary

- **date:** 2026-06-22
- **slice:** Reconcile Directional-Contrast Cohort v1 -- Settlement Completion Pass
- **cohort size:** 10 (AgentRunKey 200013-200022)
- **reconciled count:** 0
- **pending count:** 10 (all Preview/Scheduled)
- **pre-state totals:** AgentRunOutcomes 35 / AgentRunEvaluations 35
- **post-state totals:** unchanged 35 / 35 (nothing written)
- **correct / incorrect / inconclusive:** n/a this pass (0 reconciled)
- **directional-only correct / incorrect:** n/a this pass
- **home-lean vs away-lean result:** n/a this pass -- produced on the settlement-completion pass

Reason: this reconcile pass ran the same day the cohort was generated, before first pitch. All 10 games
settle tonight; expect Final ~2026-06-23T04:00Z+. Timing block only -- no identity, tenant, source, or
matcher issue.

## 2. Settlement Manifest

Source: MLB StatsAPI `schedule` (sportId=1, hydrate=linescore), checked 2026-06-22. All 10 games
`abstractGameState=Preview` / `detailedState=Scheduled`; `totalGamesInProgress=0`. Final score / winning
side / OutcomeStatus pending. Structured LeanSide is the persisted denormalized column (grade target).

- **200013** a6b5433e | gamePk 824261 | Yankees at Tigers | LeanSide **away** | first pitch 18:10Z | StatsAPI Preview/Scheduled | gate pending | manual / statsapi:824261
- **200014** abb5433e | gamePk 822964 | Royals at Rays | LeanSide **home** | 18:40Z | Preview/Scheduled | pending | statsapi:822964
- **200015** b2b5433e | gamePk 823851 | Rangers at Marlins | LeanSide **home** | 18:40Z | Preview/Scheduled | pending | statsapi:823851
- **200016** b8b5433e | gamePk 822720 | Phillies at Nationals | LeanSide **null** | 18:45Z | Preview/Scheduled | pending | statsapi:822720
- **200017** b9b5433e | gamePk 822800 | Astros at Blue Jays | LeanSide **home** | 19:07Z | Preview/Scheduled | pending | statsapi:822800
- **200018** beb5433e | gamePk 823613 | Cubs at Mets | LeanSide **away** | 19:10Z | Preview/Scheduled | pending | statsapi:823613
- **200019** c0b5433e | gamePk 824502 | Brewers at Reds | LeanSide **null** | 19:10Z | Preview/Scheduled | pending | statsapi:824502
- **200020** c7b5433e | gamePk 824585 | Guardians at White Sox | LeanSide **away** | 19:40Z | Preview/Scheduled | pending | statsapi:824585
- **200021** cab5433e | gamePk 823692 | Dodgers at Twins | LeanSide **home** | 19:40Z | Preview/Scheduled | pending | statsapi:823692
- **200022** d1b5433e | gamePk 823040 | Diamondbacks at Cardinals | LeanSide **home** | 19:45Z | Preview/Scheduled | pending | statsapi:823040

Expected evaluation interpretation when Final (contract semantics): home lean + home winner = correct;
home lean + away winner = incorrect; away lean + away winner = correct; away lean + home winner =
incorrect; null lean (200016, 200019) = inconclusive.

## 3. Reconciliation Results

None this pass. No `POST /api/agent-runs/reconcile` submitted. No outcome row, no evaluation row, no
in-progress/0-0 score posted, no outcome fabricated.

## 4. Cohort Integrity (read-only, confirmed)

All 10 unchanged since capture and reconcile-eligible once Final: `Status=completed`, artifact
`sports_decision_artifact_v3`, identity-bearing (`mlb_statsapi` + gamePk), active (`ExclusionReason=null`),
`TenantKey=1`, 0 outcome / 0 eval each. Lean distribution home 5 / away 3 / null 2 (per capture). When
this cohort settles it must be graded on the denormalized `AgentRun.LeanSide`, never prose; the 2 null
leans (200016, 200019) will evaluate inconclusive by contract.

## 5. Diagnostic Read

Outcome-based questions cannot be answered this pass (0 reconciled). They are deferred to the
settlement-completion pass:

- Did enriched directional runs beat the home-team base rate? -- pending settlement.
- Did away leans perform differently from home leans? -- pending settlement (the 3 away leans 200013 /
  200018 / 200020 are the diagnostic core).
- Did confidence discriminate inside directional runs? -- pre-settlement observation only: confidence is
  flat 0.75 across all 8 directional runs, so even after settlement it cannot discriminate *within* this
  cohort; outcomes will only show whether 0.75 is over- or under-confident overall.
- Did named-risk grounding align with outcomes? -- pending; note 7/10 Ungrounded at capture.
- Did null/no-depth runs behave as expected? -- they took a null lean at reduced confidence
  (0.45 / 0.54), as designed; they will evaluate inconclusive.
- Does this justify confidence tuning yet? -- **no.** No reconciled outcomes exist for this cohort, and
  the hard rule stands: no confidence/prompt/posture/buyer/source-depth change until a settled,
  directionally balanced read shows a discrimination signal.

## 6. Next Recommended Slice

**Additional Directional-Contrast Cohort Capture v2** is NOT yet warranted -- first settle and reconcile
this cohort. The immediate next action is the **settlement-completion pass of this same slice** (rerun
after all 10 are Final, ~2026-06-23T04:00Z+): reconcile the 10 `(mlb_statsapi, gamePk)` keys, grade on
structured LeanSide, expect totals 35/35 -> 45/45, then report home-lean vs away-lean accuracy and
whether enriched directional runs beat the home base rate. After that, **Reconciled-Outcome Calibration
Read v2** consumes the settled cohort. Choose between Named-Risk Grounding Enforcement and
Confidence-to-Evidence Coupling only after that read shows where the settled misses concentrate.

## 7. Verification

- **Repo state:** dai main 2d7ade2 clean synced; dai-vault main a8783b1 (capture commit) pushed to origin
  before this pass, then this wait-only doc committed on top.
- **Services:** docker `devcore-sql` Up; DevCore.Api `:5007` health 200. agent-service not needed (no
  generation).
- **Pre-state (sqlcmd):** AgentRunOutcomes 35 / AgentRunEvaluations 35; runs 200013-200022 each
  identity-confirmed (`mlb_statsapi`+gamePk, active, completed) with 0 outcome / 0 eval.
- **Settlement source:** MLB StatsAPI schedule, all 10 Preview/Scheduled, 0 in progress -> gate not met.
- **Endpoint calls:** none (gate not met).
- **Post-state:** unchanged 35 / 35. No rows written.
- No code/model/confidence/posture/lean/buyer/source-depth/reconciliation-logic change.
