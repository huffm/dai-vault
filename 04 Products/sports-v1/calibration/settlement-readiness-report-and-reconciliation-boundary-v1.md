# Settlement Readiness Report + Reconciliation Boundary Hardening v1

**date:** 2026-06-24
**status:** SYSTEM-IMPROVEMENT LANE (measurement hygiene). Docs-only. No reconciliation, no outcome/evaluation
writes, no decision-engine or tuning change. Read-only DB + StatsAPI inspection only. Corpus unchanged 47/47.
**type:** prepares (does not perform) tomorrow's reconciliation. Ran through dai-slice-runner + dai-skill-router
gate + dai-grill-with-vault + superpowers:verification-before-completion.
**purpose:** make the 2026-06-25 reconciliation pass lower-risk and less manual via (1) explicit lane boundaries,
(2) a classification taxonomy with rules, (3) a pre-resolved staged-run inventory, (4) a reconciliation checklist.

## 1. Lane boundaries (hard separation)

| lane | does | this slice |
|---|---|---|
| **Reconciliation** | check authoritative finals; verify identity; classify final/pending/excluded; WRITE AgentRunOutcomes + Evaluations; measure DAI vs actual/market/home/chance | NOT this slice |
| **System improvement** | deterministic support around reconciliation: make readiness visible before writes; reduce manual classification risk; document exclusion/pending/duplicate rules; improve handoff; low-risk docs/read-only reports/tests | **THIS slice** |
| **Tuning** | change prompts, model settings, lean rules, confidence logic, source-sufficiency thresholds, advertised strength, buyer copy, market weighting, artifact semantics | FORBIDDEN here and in reconciliation |

Reconciliation writes rows; system improvement makes the write safer; tuning changes what the system decides. They
never blur. A readiness report is not a reconciliation, and reconciliation is not tuning.

## 2. Readiness status taxonomy + classification rules

Each staged/pending run is classified into exactly one **status** (deterministic, from authoritative state):

- `final_ready` -- game is Final in StatsAPI with a numeric score; run is active, identity-bearing, not yet
  reconciled. ELIGIBLE.
- `pending_not_final` -- game Scheduled / Pre-Game / In Progress. NOT eligible (no guessing partial scores).
- `postponed_or_suspended` -- game Postponed/Suspended; no final outcome. NOT eligible.
- `delayed` -- Delayed Start; treat as pending until Final. NOT eligible.
- `already_reconciled` -- an AgentRunOutcome or Evaluation already exists. SKIP (do not double-write).
- `excluded` -- `ExclusionReason` set. SKIP (stays excluded).
- `invalid` -- malformed/incomplete run (e.g. no artifact); `ExclusionReason='invalid'`. SKIP.
- `duplicate_risk` -- >1 active run shares the same `(provider, gamePk)`. Reconcile per-run by run id (never by
  identity -> MultipleMatches); flag the shared game so a later read does not double-count it.
- `identity_risk` -- missing/uncertain `ExternalGameId` or `SourceProvider`. MANUAL REVIEW before any write.
- `missing_market_baseline` -- no captured market favorite for the run; still reconcilable for actual/home/chance,
  but the DAI-vs-market comparison is unavailable for it.
- `missing_artifact` -- no persisted artifact / lean. MANUAL REVIEW; do not write.
- `unknown_requires_manual_review` -- anything not cleanly classified above.

Overlay flags (not mutually exclusive with the status): `duplicate_risk`, `missing_market_baseline`,
`identity_risk` can co-occur with `pending_not_final` / `final_ready`. The **eligibility** rule:
reconcile only `final_ready` AND not `already_reconciled`/`excluded`/`invalid`/`identity_risk`/`missing_artifact`.

DAI-market relationship (per run, from the captured de-vigged consensus favorite):
`market_aligned` (DAI lean == favorite) | `market_disagreement` (DAI lean != favorite) |
`null_abstention` (DAI null lean) | `market_missing` (no baseline captured).

## 3. Staged-run inventory (read-only, resolved 2026-06-24 ~17:26Z)

Stable fields resolved now; `status` is today's value (all pending pre-settlement) and flips to `final_ready` per
run once StatsAPI shows Final tomorrow. Source of identity: `mlb_statsapi` + gamePk for all. Outcome/Eval: none.

| run | gamePk | matchup (away @ home) | game date | DAI lean | mkt fav | DAI-market | outcome/eval | excl | activeSameGame | today status + flags |
|---|---|---|---|---|---|---|---|---|---|---|
| d379433e | 823850 | Rangers @ Marlins | 2026-06-24 | away | away | aligned | no/no | - | 1 | pending_not_final |
| d879433e | 823613 | Cubs @ Mets | 2026-06-24 | home | home | aligned | no/no | - | 2 | pending_not_final + duplicate_risk |
| da79433e | 824583 | Guardians @ White Sox | 2026-06-24 | away | away | aligned | no/no | - | 1 | pending_not_final (delayed start) |
| db79433e | 824341 | Red Sox @ Rockies | 2026-06-24 | away | away | aligned | no/no | - | 1 | pending_not_final |
| de79433e | 824018 | Orioles @ Angels | 2026-06-24 | home | home | aligned | no/no | - | 1 | pending_not_final |
| e379433e | 824259 | Yankees @ Tigers | 2026-06-24 | home | home | aligned | no/no | - | 1 | pending_not_final |
| e979433e | 822965 | Royals @ Rays | 2026-06-24 | home | home | aligned | no/no | - | 1 | pending_not_final |
| ee79433e | 823367 | Mariners @ Pirates | 2026-06-24 | home | home | aligned | no/no | - | 1 | pending_not_final |
| f579433e | 822719 | Phillies @ Nationals | 2026-06-24 | away | away | aligned | no/no | - | 1 | pending_not_final |
| f779433e | 822798 | Astros @ Blue Jays | 2026-06-24 | home | home | aligned | no/no | - | 1 | pending_not_final |
| fe79433e | 824500 | Brewers @ Reds | 2026-06-24 | home | away | **disagreement** | no/no | - | 1 | pending_not_final (scarce-signal) |
| 037a433e | 823691 | Dodgers @ Twins | 2026-06-24 | away | away | aligned | no/no | - | 1 | pending_not_final |
| 047a433e | 823041 | Diamondbacks @ Cardinals | 2026-06-24 | home | home | aligned | no/no | - | 1 | pending_not_final |
| 077a433e | 823284 | Braves @ Padres | 2026-06-24 | null | away | **null_abstention** | no/no | - | 1 | pending_not_final (scarce-signal) |
| beb5433e | 823613 | Cubs @ Mets (200018) | 2026-06-22 | away | (home) | disagreement / **market_missing** | no/no | - | 2 | pending_not_final + duplicate_risk + missing_market_baseline |

Counts: 15 staged, 0 reconciled, 0 excluded among staged (all 13 prior exclusions remain excluded and are NOT in
this list). Scarce-signal runs to grade first: fe79433e (DAI-market disagreement), 077a433e (null abstention), and
the 823613 pair. Duplicate-risk pair: **d879433e + beb5433e share gamePk 823613** with opposite leans.

## 4. The 823613 duplicate-risk (explicit guidance)

beb5433e (200018, captured 06-22 for the postponed Cubs@Mets) and d879433e (cohort-v2, captured 06-24) are two
independent DAI reads of the **same** rescheduled game (gamePk 823613), with opposite leans (away vs home). Rule
for tomorrow: reconcile **both per-run by run id** (`POST /{id}/outcome`), never by identity (would return
MultipleMatches). They are the same game outcome, so a later calibration read must NOT treat them as two
independent observations of game direction -- keep them in their own cohorts and note the shared gamePk. If only
one should count, the operator decides explicitly; this slice does not pre-exclude either.

## 5. Tomorrow's reconciliation checklist (DIRECTIONAL-CONTRAST SETTLEMENT PASS V1)

Gate: run after **2026-06-25T12:00:00Z or later**.

1. Verify repo state; bring up stack (devcore-sql, DevCore.Api :5007); StatsAPI reachable.
2. Confirm pre-state corpus (expect 47/47 by sqlcmd). Confirm the 13 prior exclusions are still excluded.
3. For each of the 15 staged gamePks, re-query StatsAPI finality. Flip `pending_not_final` -> `final_ready` only
   when `detailedState=Final` with a numeric linescore. Leave Postponed/Suspended as not-eligible.
4. For each `final_ready` + not-already-reconciled run: `POST /api/agent-runs/{id}/outcome` with the StatsAPI score
   (home/away/outcomeStatus), then `GET /{id}/evaluation`. Per-run by run id only.
5. Honor `duplicate_risk` (823613): reconcile beb5433e and d879433e by run id; record both, flag the shared game.
6. Do NOT post for any pending/postponed/excluded/identity-risk/missing-artifact run. Never guess a score.
7. Post-state: confirm outcomes/evaluations grew by exactly the number of runs reconciled; record per-run
   id/matchup/lean/market-favorite/outcome/eval.
8. Measure DAI vs actual / market-favorite / home / chance; isolate the disagreement (fe79433e, beb5433e) and null
   (077a433e) games. Keep regimes separate; do not pool the 823613 pair as independent.
9. No tuning. Document a settlement doc + update current-slice. Commit docs (+ operational data) only; do not push.

## 6. What did not change

No reconciliation performed; no AgentRunOutcomes/Evaluations rows written (corpus 47/47, verified before and after);
no winners inferred; no partial scores read as performance; no decision-engine/prompt/model/confidence/lean/source-
threshold/advertised-strength/buyer-copy/artifact-semantics change; no new slate captured; no code change (docs
only); no push.

## 7. Deferred (noted, not built)

A read-only `settlement-readiness` script or endpoint that regenerates section 3 each pass was considered and
deferred: the stable fields are already resolved here and finality is a one-line StatsAPI re-check, so a committed
live-DB tool is not yet clearly justified. Revisit if the multi-slate capture cadence makes per-day regeneration
worth a tested helper (would live read-only under `scripts/dev/sports/`, no writes, with unit-tested classification).
