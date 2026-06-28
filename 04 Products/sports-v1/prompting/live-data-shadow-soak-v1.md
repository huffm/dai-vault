# Live-Data Shadow Soak v1 -- Readiness Report

**status:** active doctrine (READINESS report -- live-data soak NOT executed; harness ready)
**date:** 2026-06-28

## purpose

Record the outcome of attempting a live-data shadow-parallel soak for MLB, and the standing readiness state: live-data access
was unavailable through any legitimate path, so no live soak was run. A live-data MODE was added to the existing soak harness
and verified offline; this doc captures the access blocker, the harness readiness, and the exact command/environment to run
the live soak when access exists. Per the slice access rule, nothing here simulates or overclaims live data.

## problem it solves

[[shadow-parallel-cohort-soak-v1]] proved prompt byte-equivalence across all 9 MLB regimes on REPRESENTATIVE fixtures. The
open question before any registry-authoritative routing is whether REAL MLB input shapes expose gaps representative fixtures
miss (partial evidence, regime distribution, formatting drift, assembly/sink failures, equality mismatches). This slice makes
the harness able to consume real analyzer inputs and records that live data was not reachable here.

## strategic fit

The prompting migration track (Phase 4 -> 4.1 -> Migration Readiness -> Shadow-Parallel Routing -> Cohort Soak -> this) is the
controlled path toward letting an approved prompt registry eventually drive the model input. Each step is evidence-gated and
reversible. Live-data evidence is the gate this slice was meant to clear; it remains open due to access, not due to a defect.

## mental model

The soak harness is a measurement instrument. Representative mode runs a built-in 9-regime cohort; LIVE-DATA mode runs a cohort
loaded from real analyzer request records. Both feed the exact default-off shadow sidecar (`run_mlb_shadow_validation`) the
analyzer runs when enabled -- no model call, no second artifact. The instrument is ready; the live sample was not available.

## what it is

- A LIVE-DATA mode added to `app/services/shadow_cohort_soak.py`:
  - `cohort_from_request_records(records)` -- builds the cohort from real analyzer request dicts, validated by the SAME
    `SportsAnalysisRequest` pydantic model the live endpoint uses (faithful real input shapes); rejects non-mlb / malformed
    records (fails loud, never silently shrinks the cohort).
  - `load_cohort_from_requests(path)` -- reads a json array or jsonl file of request records; does no network/db i/o itself.
  - `_is_partial_evidence(starter, market)` -- detects, on the assembly-failure path, real input shapes the current overlay
    templates cannot represent (enriched starter with only one season-form side; a depth market missing a depth metric), so a
    real partial-evidence run is classified as the known `partial_evidence_unrepresentable` gap, not mislabeled a bug.
  - `SoakSummary.unrepresented` -- records partial-evidence shapes with detail (the assembly error names the missing slots) to
    design the next overlay; these do NOT break `clean` and are NOT in `failures`.
- A runnable `scripts/run_shadow_cohort_soak.py [--requests <file>] [--sink <file>]` -- `--requests` selects live-data mode;
  default is representative; exit 0 iff GO (clean AND whole cohort validated).

## what it is not

- NOT registry-authoritative routing. NOT a model call or a second artifact. NOT a buyer change. NOT settlement / calibration
  scoring / outcome claims. NOT a live-data result -- no live data was run.

## approved uses

- Run the representative soak any time as a harness health check.
- When DB/network/local service access exists, produce a real requests file and run live-data mode (see example usage).

## disallowed uses

- Do not present synthetic/representative request files as live-data results.
- Do not enable the validator globally; enable only via the explicit soak config.
- Do not change prompt templates or the manifest to "make a real shape pass" inside this slice -- if a real gap appears, stop
  and scope a partial-evidence overlay slice.

## workflow impact

The soak is the evidence step before `Registry-Authoritative Prompt v1`. Until a live-data soak runs clean (or any gap is
overlay-fixed), registry-authoritative routing stays gated.

## truth hierarchy

The soak is a measurement layer, never an authority. Source, tests, and the live prompt remain authoritative; a GO verdict is
necessary-not-sufficient evidence, not permission to route.

## source or vault references to verify

- Live-data mode + detector: `services/agent-service/app/services/shadow_cohort_soak.py`
  (`cohort_from_request_records`, `load_cohort_from_requests`, `_is_partial_evidence`, `_classify`).
- Request shape: `services/agent-service/app/models/sports.py` `SportsAnalysisRequest`.
- Sidecar reused: `services/agent-service/app/services/shadow_validation.py` `run_mlb_shadow_validation`.
- Tests: `services/agent-service/tests/test_shadow_cohort_soak.py` (15 cases, incl. 6 live-data).
- Detector accuracy verified against `app/services/migration_readiness.py` + `app/prompting/manifest.json` requiredSlots
  (enriched needs both `*_starter_form`; backed_depth needs `median_home_prob`/`disagreement_pct`/`disagreement_strength`/
  `median_total`) -- adversarial check found exact match (6/6 cases).

## example usage

Live-data run (when access exists), from `services/agent-service`:

```
# 1. produce a real requests file (out-of-band). either:
#    a. start the .net platform-api + devcore-sql, drive the MLB retrieval path, and capture the
#       SportsAnalysisRequest payloads the platform posts to the agent-service; or
#    b. export persisted agent-run request inputs for RunType=sports.matchup.analysis to a jsonl file,
#       one {competition,homeTeam,awayTeam,gameDate,mlbStarterContext?,baseballMarketContext?} per line.
# 2. run the live-data soak against a scratch (out-of-repo) sink:
python scripts/run_shadow_cohort_soak.py --requests <reqs.jsonl> --sink <scratch>/live_soak.jsonl
# exit 0 = GO (clean and whole cohort validated). inspect the json report's counts + regimes + unrepresented.
```

## acceptance criteria

- Representative soak: GO, 18/18 matched+captured (verified this slice).
- Live-data mode: loads real request shapes, counts every bucket, auto-detects partial evidence, fails loud on bad records
  (verified offline, 6 tests).
- Full agent-service suite green; manifest OK; `sports_analyzer.py` and `.NET` unchanged (verified).

## risks and failure modes

- **Representative != live (open gate).** Byte-equivalence is proven only on representative inputs; real data may expose
  partial-evidence shapes (counted, non-drift), regime-distribution skew, or -- if any mismatch/assembly/sink/error appears --
  a real blocker that forbids registry-authoritative routing (slice rule 9).
- **Detector duplicates representability rules.** `_is_partial_evidence` re-encodes the template requirements; it is consulted
  only on the failed-assembly path and was verified exact, but a future overlay change must keep it in sync (or consolidate).
- **Access fragility.** The live run depends on the platform retrieval path or a persisted-run export; neither is wired into
  this harness (by design -- the loader is file-based, no network/db).

## deferred decisions

- DB persistence of provenance (still deferred; JSONL sink only).
- `.NET sourceDepth -> dataRegime` ownership (still python).
- A partial-evidence overlay (only if live data shows these shapes are common).
- Registry/builder caching for enabled mode.

## live-data soak result (this slice)

- **Live-data access available?** NO. platform-api not running (no :5000/:5028/:7028); vault calibration artifacts are OUTPUT
  decision artifacts with no input contexts (no starter/market); no persisted request-input files exist on disk
  (`grep mlbStarterContext|baseballMarketContext` over the workspace = 0 json hits); devcore-sql is up but extracting inputs
  would require harvesting DB credentials, which was correctly blocked by the sandbox and is not an approved path.
- **Data source used:** none (no live soak executed). Harness verified offline with synthetic request SHAPES (clearly labeled,
  not live) and the representative cohort.
- **Cohort size / regime distribution / counts / mismatches / partial evidence:** n/a for live data (not run). Representative
  harness health: 18 games, all 9 regimes x2, 18 matched, 18 captured, 0 failures, GO.
- **Enablement method:** explicit `ShadowValidationConfig(enabled=True, sink_path=...)` for the soak only; validator default-off
  otherwise.
- **Sink path:** scratch / out-of-repo; no JSONL artifact committed.
- **Why live behavior remains unchanged:** `sports_analyzer.py` and `.NET` unchanged this slice (git-confirmed); only the soak
  harness/script/tests changed (additive); the default-off sidecar and live prompt authority are untouched.
- **Exact tests run:** `pytest tests/test_shadow_cohort_soak.py -q` -> 15 passed; `pytest -q` -> 287 prior +6 = **293 passed,
  0 failed**; `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0; representative soak -> GO 18/18.

## related docs

- [[shadow-parallel-cohort-soak-v1]] (representative soak), [[live-prompt-routing-shadow-parallel-v1]],
  [[live-migration-readiness-v1]], [[prompt-provenance-persistence-v1]], [[prompt-provenance-and-manifest-integrity-v1]].

## recommended next slice

**Execute Live-Data Shadow Soak (when access exists):** bring up platform-api + devcore-sql (or export persisted request
inputs), produce a real requests file, and run `python scripts/run_shadow_cohort_soak.py --requests <file>`. If
`partial_evidence_unrepresentable > 0`, scope a partial-evidence overlay slice before routing; if any mismatch/assembly/sink/
error, do not route. Only on a clean live GO proceed to **Registry-Authoritative Prompt v1** (registry prompt becomes model
input for proven-clean regimes, live builder as fallback + equality guard). Then add the registry/builder cache and decide
`.NET sourceDepth -> dataRegime` ownership. Defer DB persistence, protocol-as-execution, buyer UI.
