# Cognitive Protocol Run Export v1 -- Calibration Note

**date:** 2026-05-18
**export:** `2026-05-18-cognitive-protocol-run-export.csv` (same folder)
**status:** generated artifact columns filled; outcome columns intentionally blank, pending game results.

---

## 1. batch purpose

Generate a small batch of fresh sports analyses and export them to a spreadsheet-friendly
file so the generated read can later be compared against actual game outcomes. This is the
data-collection half of an outcome-reconciliation loop: the run is recorded now, the result
is recorded after the games finish, and the two are compared. NBA was prioritized because
games are active; MLB filled the rest because NBA volume alone was too low.

## 2. run count

8 runs, all `status: completed`. No failed runs.

| competition | runs |
|---|---|
| nba | 2 |
| mlb | 6 |
| **total** | **8** |

Run ids are in the `run_id` column of the CSV.

## 3. leagues included

- **NBA (2):** San Antonio Spurs at Oklahoma City Thunder (2026-05-18), Cleveland Cavaliers
  at New York Knicks (2026-05-19). Only 2 NBA games were upcoming even in a 14-day window --
  the league is in deep playoffs, so volume is naturally thin.
- **MLB (6):** six 2026-05-18 games, selected as the first 6 of 14 upcoming games.
  Included to top the batch up into the 5-10 target range, per the volume rule.
- **NFL:** skipped. The provider returned 0 upcoming NFL games (offseason), so there was
  no naturally available schedule data to use.

## 4. startup or test issues

- `start-sports-dev.ps1` launched all three services cleanly. Agent service came up on
  port 8000 within ~4 seconds.
- **First smoke test: 4/5 passed.** Check 5 (full chain via the .NET platform API) failed
  with a SQL Server connection timeout.
- **Root cause (no code change):** the `devcore-sql` Docker container
  (`mcr.microsoft.com/mssql/server:2022-latest`) was not running. There is no `MSSQLSERVER`
  Windows service -- the database engine is containerized. The container resumed once the
  Docker daemon was active; after waiting for SQL Server to finish initializing and
  confirming the `devcore` database exists, the smoke test was re-run.
- **Second smoke test: 5/5 passed.** The full chain (`POST /api/agent-runs` -> SQL row
  written) was confirmed before any paid runs were created.
- No runtime code and no scripts were changed. The only issue was environment state
  (a stopped container), not a script or code defect.

## 5. observed CognitiveProtocol behavior

- **All 8 runs are v2** (`artifactVersion: sports_decision_artifact_v2`). `cognitiveProtocol`
  is present on every run (`cognitive_protocol_present = yes` for all 8).
- **Legacy `cognitivePhases` still present** on all 8 runs alongside the canonical block --
  the dual-emit compatibility model is intact. The artifact endpoint also returns the
  `protocolView` projection.
- **`position` equals `read_stance` (posture) on all 8 runs** -- every run resolved to
  `monitor`. This matches doctrine: `Decide.Position` preserves the validated posture string
  exactly, it is not a separate pick.
- **`stress` (canonical `Discern.Stress`) populated on all 8 runs**, single-sourced and
  readable -- no concatenation artifacts observed.
- **`probe` populated on 2 of 8 runs** -- both NBA runs, with the doctrinal sharp_public
  template. See section 7.

## 6. repeated weaknesses

- **High confidence on thin evidence (5 of 8 runs).** The harness flag
  `confidence_high_for_partial_evidence` fired on both NBA runs (evidence richness 2,
  confidence 0.72) and 3 of 6 MLB runs (evidence richness 1, confidence 0.70-0.75).
  This is the single dominant pattern across the batch and the main thing outcome
  reconciliation needs to test.
- **Zero posture variance.** All 8 runs resolved to `monitor` -- no `play`, no `pass`.
  Whether that uniformity is correct caution or calibration flatness cannot be judged
  without outcomes.
- **MLB signal coverage is very thin.** Every MLB run grounded exactly one signal
  (`starting_pitching`), `evidence_richness = 1`, `missing_signals = none`, and no
  `signal_followups`. MLB currently has no market, sharp/public, or rest signal in the
  pipeline at all -- the strongest and weakest signal are the same single signal.
- **MLB analyzer references ungrounded signals (2 of 6 runs).** Orioles@Rays and
  Reds@Phillies carry quality warnings: `signals_used contains bullpen / lineup_form /
  ballpark but ... was not grounded`. The model narrated signals it did not actually have.
- **Minor prompt-quality flags.** NBA: `frame_missing_rest_context` on 2/2,
  `interrogate_unsupported_claim` on 1/2. MLB: `counter_case_generic`,
  `what_would_change_contains_filler`, and `interrogate_unsupported_claim` on 1 run each.
  These are scattered, not dominant.
- **NBA sharp_public gap persists.** Both NBA runs are missing `sharp_public`
  (`provider_returned_empty_odds`) -- the same gap seen across the 10 prior NBA calibration
  reports in the parent folder.

## 7. did Probe Population v1 improve artifact review?

Yes, where follow-up data exists -- and it behaved correctly where it does not.

- **NBA (2/2): Probe populated.** Both runs are missing `sharp_public`, which produces a
  missing-primary follow-up record, which the deterministic Probe builder turns into the
  doctrinal template: *"Sharp/public signal missing; market read relies on price movement
  only."* This is a one-line, grounded statement of the coverage gap -- a reviewer sees the
  artifact's main weakness without reading the full signal section. That is a real
  improvement.
- **MLB (0/6): Probe null, and correctly so.** MLB runs have `missing_signals = none` and
  no `signal_followups`, so there is no missing-primary record for the Probe builder to act
  on. `not_recorded` is the correct output, not a defect. Probe never fabricated injury,
  form, or travel content.

**Verdict:** Probe Population v1 works as designed. Its usefulness is bounded by upstream
signal/follow-up coverage -- it adds value on runs with real diagnosed gaps (NBA) and stays
silent on runs with none (MLB). The MLB blank is an MLB signal-pipeline gap, not a Probe
problem.

## 8. recommended next slice

**Primary: Outcome Reconciliation v1.** This batch exists to be scored against results.
The CSV's outcome columns are built and waiting. The dominant weakness --
high confidence (>=0.70) on thin evidence (richness 1-2) on 5 of 8 runs -- cannot be acted
on until there are actual outcomes to test it against. The two harness reports separately
recommend *Cognitive Prompt Tightening v1.5* (NBA) and *Confidence Calibration Rules v1*
(MLB); combined across all 8 runs the `confidence_high_for_partial_evidence` pattern (5/8)
points at confidence calibration. But confidence calibration is unfalsifiable without
outcome data. Reconciliation is the prerequisite step; do it first, then revisit
*Confidence Calibration Rules v1* with evidence.

**Secondary: MLB signal coverage expansion.** MLB grounds only `starting_pitching` and the
analyzer narrates signals (bullpen, lineup_form, ballpark) it never received. Bringing MLB
to parity with NBA's market/rest grounding would make MLB runs meaningfully comparable and
would give the MLB Probe something to work with.

The artifact/protocol layer itself (v2 contract, dual-emit, Probe v1, canonical Stress,
Position) showed no defects in this batch -- it does not need to be reopened.

---

## appendix: CSV column notes

The CSV mixes direct artifact fields with three **derived** columns. Derived columns use
only recorded runtime data or a documented code rule -- no values were inferred from
narrative text and no missing values were fabricated.

| column | source | type |
|---|---|---|
| `confidence_band` | **derived** from `confidence` using the documented rule in `SportsEvaluator.cs:52-54`: `>=0.70` high, `>=0.50` medium, else low. (This reproduces the evaluator's own `ConfidenceBand`, which is not surfaced on the artifact DTO.) | derived |
| `strongest_signal` | **derived** from `signalAvailability`: the signal(s) with the highest recorded `quality` (strong > usable > unknown > unavailable). Annotated with the quality value. | derived |
| `weakest_signal` | **derived** from `signalAvailability`: the signal(s) with the lowest recorded `quality`. On single-signal runs (all MLB) strongest and weakest are the same signal. | derived |
| `lean_side` | **derived** by matching the recorded `lean` text against each team's full name, nickname, then city. `none` means the lean text named no team (e.g. "Signals are split"). | derived |
| all other generated columns | direct from the artifact endpoint or the run record | direct |
| `final_score_home`, `final_score_away`, `actual_winner`, `actual_cover_result`, `actual_total_result`, `outcome_notes`, `post_result_review_status` | left blank -- outcome reconciliation inputs | blank |

`not_recorded` is used where a field has no value in the run output (e.g. MLB `probe`).
`none` is used where a list field is legitimately empty (e.g. `missing_signals`,
`signal_followups`, `quality_warnings`).

## appendix: related generated files

The calibration harness (`run-artifact-calibration.ps1`) was used to create the runs. As a
byproduct it also wrote, into the parent `calibration/` folder:

- `20260518-1001-nba-calibration.md` and `20260518-1010-mlb-calibration.md` -- harness
  reports.
- `artifacts/20260518-1001-nba-*.json` and `artifacts/20260518-1010-mlb-*.json` -- 8 raw
  artifact JSON files.

These are standard harness output, not part of this export, and are left in place.
