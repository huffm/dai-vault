# Reconciliation and Fresh Batch Validation -- 2026-06-23 v1

**date:** 2026-06-23
**status:** operational validation (no code change). Stack brought up, fresh MLB batch generated, advertised-strength server authority spot-checked live. Reconciliation execution deferred (no authoritative outcomes available; harness never guesses).
**type:** validation slice run through dai-slice-runner + dai-test-discipline (service/endpoint checks). No dai-code-reviewer (no code changed).

## purpose

Pause Advertised-Strength Frontend Adoption v1, reconcile eligible completed games, run a small fresh batch of current games, and record operational state so the frontend adoption resumes with cleaner evidence. A secondary aim: spot-check Advertised-Strength Server Authority v1 (`a499b80`) on live artifacts.

## services used

- devcore-sql (docker container): started this slice (Docker Desktop launched first); Up on :1433, "ready for client connections".
- FastAPI agent-service (:8000): `{"status":"ok"}`.
- platform-api / DevCore.Api (:5007): health 200. Built from HEAD `a499b80` so the advertised-strength derive-on-read is live.
- Angular sports-app: NOT started (no UI validation this slice; frontend adoption deferred).

## repo state

- `<DAI_REPO_ROOT>`: main `a499b80`, clean (no source change this slice), 1 commit ahead of origin (unpushed, preserved).
- `<DAI_VAULT_ROOT>`: main `51b0f75` before this slice, 1 ahead of origin (unpushed, preserved). This report + current-slice are the only changes.

## reconciliation summary

Inventory via `GET /api/agent-runs/recent` + `GET /api/agent-runs/{id}/evaluation`:

- runs inspected: 25 (all MLB, all run-status `completed`), from 2026-06-19 and 2026-06-22.
- already reconciled (have an evaluation): 6.
- not reconciled: 19.
- **reconciled this slice: 0.**

Why 0: the existing reconciliation path (`POST /api/agent-runs/{id}/outcome` -> `POST /reconcile`, and `reconcile-calibration-outcomes.ps1`) requires **real, human-supplied final scores** and never guesses an outcome (hard rule: do not guess outcomes). No authoritative results were available to this session, so no outcomes were posted. Additionally, several of the 19 are duplicate test-matchup reruns from the prior near-close-capture testing (e.g. New York Yankees @ Detroit Tigers x6 on 2026-06-22); reconciling those by identity would return `MultipleMatches` and they must be excluded (`POST /{id}/exclude`) before reconciliation.

- eligible and reconciled this slice: none.
- not yet eligible / blocked: the 19 await (a) authoritative final scores in a results-input CSV and (b) de-duplication via exclusion for the repeated matchups.

## fresh batch summary

Selection: 4 distinct MLB games for 2026-06-23 from `GET /api/competitions/mlb/upcoming`, chosen to avoid the prior test matchups. Generated via `POST /api/agent-runs` (RunType `sports.matchup.analysis`); all completed.

| agent run id | matchup (away @ home) | artifact ver | leanSide | confidence | evidenceRichness | sufficiency | advertisedStrength | rawBand | reason | grounded | depth |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 6a2d433e | Seattle Mariners @ Pittsburgh Pirates | v3 | away | 0.75 | 2 | moderate | High | High | (none) | starting_pitching, market | sp=enriched, market=enriched |
| 702d433e | Boston Red Sox @ Colorado Rockies | v3 | away | 0.80 | 2 | moderate | High | High | (none) | starting_pitching, market | sp=enriched, market=enriched |
| 722d433e | Baltimore Orioles @ Los Angeles Angels | v3 | home (!) | 0.70 | 2 | moderate | High | High | (none) | starting_pitching, market | sp=enriched, market=enriched |
| 752d433e | Atlanta Braves @ San Diego Padres | v3 | (null) | 0.45 | 1 | thin | Medium | Medium | (none) | market | market=enriched, sp=none |

- failed/skipped runs: none.
- buyer-safe posture: leans are measured ("slight lean ... based on starting pitching"), no tout language; run 4 is a "no clear lean" abstention. No quality warnings fired on any run.
- model cost: each run made the single analyze model call (cost telemetry logged server-side per run, ~sub-cent each per prior smoke; not separately captured here).

## advertised-strength spot check

- /artifact includes `advertisedStrength` where expected: **yes** -- present on all 4 fresh runs and on the existing run e5b5433e (derive-on-read working).
- internal confidence unchanged: **yes** -- `confidence` is carried through unchanged; `advertisedStrength` is a separate derived field.
- moderate/rich + high -> High (not over-capped): **yes** -- runs 1-3 (conf 0.70-0.80, richness 2) correctly stay High.
- thin + non-high -> unchanged (no over-cap): **yes** -- run 4 (thin, conf 0.45) -> Medium, no reason.
- thin evidence cannot advertise as High (cap fires): **not observed live this batch** -- today's MLB slate grounds 2 signals (starting_pitching + market) -> moderate, so no thin+high run occurred. The cap (High -> Medium with reason `advertised_strength_limited_by_evidence`) is covered by 13/13 unit tests (`AdvertisedStrengthTests`) and the documented historical case (Evidence-Sufficiency Band Gate v1: richness 1, conf 0.75 -> Medium). Recommend capturing a live thin+high case opportunistically (a single-signal high-confidence MLB run).
- missing evidenceRichness fail-open: not exercised (no legacy run without richness in the current 25); covered by unit test `fails_open_when_evidence_richness_is_absent`.
- frontend adoption performed: **no** (intentional).
- contract concern: none new; the server field behaves per spec.

## issues found

1. **Direction PotentialMismatch on run 722d433e (Orioles @ Angels).** Prose + summary clearly lean the Orioles (away: "the overall backing for the Orioles is stronger"), but structured `leanSide = home` (Angels). The `ArtifactDirectionConsistency` guard **correctly flagged it as `PotentialMismatch`** (the guard works). This is a leanSide-derivation discrepancy worth a dedicated investigation. It does NOT affect advertised strength (confidence/evidence axes are independent of direction). Not fixed here (operational validation; lean derivation is out of scope and protected by hard rule 11).
2. **Reconciliation is blocked on authoritative outcomes.** Recent runs cannot be reconciled without real final scores, and the harness never guesses. The 6/22 batch also has duplicate-matchup reruns that need exclusion before reconciliation to avoid `MultipleMatches`.
3. **Cap-fire not observable live this batch** (minor): no thin+high run in the current data; logic is unit-tested.

## what did not change

No application source code, no buyer copy, no advertised-strength derivation, no confidence/lean/posture, no Tool Gateway, no source collection, no probe refresh, no reconciliation logic, no billing/auth/tenant. New operational data only: 4 fresh agent-run rows in the dev DB (expected, like prior live-verification slices). No local commits rewritten.

## recommended next slice

Advertised-Strength Frontend Adoption v1 -- render `artifact.advertisedStrength` and retire the duplicate frontend `gateAdvertisedStrength`/`confidenceBand` computation, plus an `/artifact` endpoint test. Separately worth a small slice: investigate the run 722d433e leanSide-vs-prose `PotentialMismatch`; and, when authoritative results are available, a Reconciliation Results-Input + Dedup slice (exclude duplicate reruns, then reconcile the eligible 6/19-6/22 games).
