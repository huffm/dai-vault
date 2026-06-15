# Budgeted MLB Stage 0 Generation v1

**date:** 2026-06-15
**status:** executed against local/dev with live model spend. generated the prepared 10-game MLB-only Stage 0 batch through the normal analyzer path; all 10 runs completed and are identity-bearing. no reconciliation performed (all games pending settlement). no code/schema/matcher/identity/confidence/buyer change. nothing fabricated.
**classification:** generation + identity-capture verification. NOT a reconciliation slice.

## Why this slice

Stage 0 Larger Identity-Bearing Batch v1 (dry-run) prepared an MLB-only batch and confirmed 0 unreconciled identity-bearing runs existed. This slice spends the budgeted model calls to generate that batch so the runs exist as pre-settlement, identity-bearing reconciliation candidates. NBA is offseason; WNBA is deferred (support missing) -- both untouched here.

## Services confirmed / started

- DevCore.Api: up (`GET /api/competitions` 200), `:5007`.
- dev SQL: `devcore-sql` container up, `1433` open.
- FastAPI analyzer: started this slice (`uvicorn main:app :8000`, existing venv); `GET /api/ping` 200. Required for model generation.
- dev tenant unchanged: `TenantKey=1` (dev bypass).

## Generation command

```
pwsh -File .\scripts\dev\sports\run-artifact-calibration.ps1 -Competition mlb -Days 3 -Take 10 -Force
```

Existing supported workflow; sequential, one model call at a time; hard cap 10. Dry-run was run first (no spend) to confirm the 10-game selection.

## Attempts / successes / failures

- attempted: 10. successful: 10. failed: 0.
- all returned `status: completed`. No 429/rate-limit/key failure.

## Spend estimate

10 model calls, `gpt-4o-mini`, ~2650 input + ~350-510 output tokens each. Per-call configured estimate ~$0.00061-$0.00070; **batch total ~= $0.0067 USD** (`devcore.cost` logger, `costEstimateVersion=v1`, configured estimate -- not billing truth). Well under the budget cap.

## Identity coverage

- identity-bearing runs created: **10 / 10**. Every run carries `SourceProvider=mlb_statsapi`, a statsapi `gamePk` `ExternalGameId`, populated `ScheduledStartUtc`, `Season=2026`, and ascii team refs (MLB Game Identity Capture v1 working as on the prior four MLB candidates).
- global identity-bearing run count: 4 -> **14**.
- duplicate `(SourceProvider, ExternalGameId)` groups: 0.
- artifact version: all `sports_decision_artifact_v3` (canonical/current).

## Directional usability

Authoritative source is the denormalized `AgentRun.LeanSide` column (what `RunEvaluator`/the matcher read; proven in the prior reconciliation slice).

- **directionally usable (non-null lean): 7** -- all `LeanSide=home`, posture `monitor`, confidence 0.75, evidenceRichness 1.
- **null lean (inconclusive-only): 3** -- posture `wait`, confidence 0.375-0.5, evidenceRichness 0-1.

Observation (not a defect, no fix made): the calibration-export artifact JSON leaves its `leanSide` field null on all 10 and carries the lean as prose (`lean`) instead; the structured directional value lives on the `AgentRun.LeanSide` column, which is correctly populated for the 7 monitor runs. The matcher reads the column, so reconciliation is unaffected. Flagging only as an export-projection inconsistency to watch if a future read surface consumes the export JSON's `leanSide`.

## Settlement status

All 10 games are scheduled 2026-06-15 22:40Z - 2026-06-16 02:10Z (tonight UTC), after the current clock. **All 10 are pending settlement; none is reconcilable now.** No outcome source was consulted and no reconcile call was made.

## No-write verification

`AgentRunOutcomes` and `AgentRunEvaluations` both remained at **12 / 12** after generation -- generation created zero outcome/evaluation rows, as expected. The four prior reconciled candidates are untouched.

## Candidate table

| # | run id (8) | matchup (away @ home) | gamePk | scheduled start UTC | tenant | provider | lean (AgentRun) | usable | posture | conf | evid | artifact | settled? | readiness |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | `2303433e` | Marlins @ Phillies | 823452 | 2026-06-15 22:40 | 1 | mlb_statsapi | home | yes | monitor | 0.75 | 1 | v3 | pending | pending settlement |
| 2 | `2803433e` | Royals @ Nationals | 822724 | 2026-06-15 22:45 | 1 | mlb_statsapi | home | yes | monitor | 0.75 | 1 | v3 | pending | pending settlement |
| 3 | `2a03433e` | Mets @ Reds | 824505 | 2026-06-15 23:10 | 1 | mlb_statsapi | home | yes | monitor | 0.75 | 1 | v3 | pending | pending settlement |
| 4 | `2e03433e` | Padres @ Cardinals | 823046 | 2026-06-15 23:45 | 1 | mlb_statsapi | (null) | no | wait | 0.375 | 0 | v3 | pending | null lean |
| 5 | `3403433e` | Rockies @ Cubs | 824666 | 2026-06-16 00:05 | 1 | mlb_statsapi | home | yes | monitor | 0.75 | 1 | v3 | pending | pending settlement |
| 6 | `3603433e` | Twins @ Rangers | 822887 | 2026-06-16 00:05 | 1 | mlb_statsapi | (null) | no | wait | 0.375 | 0 | v3 | pending | null lean |
| 7 | `3d03433e` | Tigers @ Astros | 824181 | 2026-06-16 00:10 | 1 | mlb_statsapi | home | yes | monitor | 0.75 | 1 | v3 | pending | pending settlement |
| 8 | `4103433e` | Angels @ Diamondbacks | 825071 | 2026-06-16 01:40 | 1 | mlb_statsapi | home | yes | monitor | 0.75 | 1 | v3 | pending | pending settlement |
| 9 | `4203433e` | Pirates @ Athletics | 824993 | 2026-06-16 01:40 | 1 | mlb_statsapi | (null) | no | wait | 0.5 | 1 | v3 | pending | null lean |
| 10 | `4903433e` | Rays @ Dodgers | 823938 | 2026-06-16 02:10 | 1 | mlb_statsapi | home | yes | monitor | 0.75 | 1 | v3 | pending | pending settlement |

Full run ids are `<8>-f36b-1410-8163-00373db4b724`. Per-run artifacts: `04 Products/sports-v1/calibration/artifacts/20260615-1237-mlb-<8>.json`; batch report: `04 Products/sports-v1/calibration/20260615-1237-mlb-calibration.md`.

## WNBA

WNBA support remains **deferred** to a separate future slice (catalog + team seed + odds sport-key + analyzer gate + readiness review). Not touched here.

## Recommended next slice

**Reconcile the 7 directionally usable MLB runs after settlement** -- once tonight's games are final, reconcile their exact `(mlb_statsapi, gamePk)` keys through `POST /api/agent-runs/reconcile` with statsapi finals (the proven path), then read back evaluations. The 3 null-lean runs will evaluate inconclusive and can be reconciled for completeness but add no directional calibration signal. This would roughly triple the directional sample (prior 3 usable + up to 7 new). Only after a non-trivial correct/incorrect body exists should Internal Calibration Read Surface v1 be considered (entry 12 stays gated). WNBA Support Setup v1 is the alternative path only if the operator intends to add WNBA.
