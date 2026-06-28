# MLB Request Input Export v1

**status:** active doctrine (export tool shipped + verified; live export NOT run -- access unavailable)
**date:** 2026-06-28

## purpose

Provide a safe, repeatable way to turn real MLB analyzer request inputs into the `--requests` file the shadow
cohort soak consumes, so the live-data shadow soak can be unblocked without changing prompt authority, making
model calls, scraping secrets, or inventing live-data evidence. This slice ships the export tool and records
that live-data access was unavailable, so no real export was produced here.

## problem it solves

[[live-data-shadow-soak-v1]] is blocked on one missing input: a file of REAL MLB analyzer request inputs. The
soak harness already accepts `--requests`, but nothing produced that file from a sanctioned source. This export
stage closes that gap with validation, output-stripping, and a summary -- decoupled from where the raw records
come from, so it carries no credentials and no retrieval logic.

## strategic fit

The prompting-migration track (Phase 4 -> 4.1 -> Migration Readiness -> Shadow-Parallel Routing -> Cohort Soak
-> Live-Data readiness -> this) is evidence-gated. Export is the input-capture step that feeds the live-data
soak, which is the evidence gate before any registry-authoritative routing.

## mental model

The export is a validation/normalization stage, not a retriever. Raw records in (from any approved source) ->
clean, input-only MLB request records out + a summary. It is the safe funnel between "some source produced
records" and "the soak validates prompt equivalence on them."

## what it is

- `app/services/mlb_soak_export.py`:
  - `extract_mlb_request_input(raw)` -> `(clean | None, regime | None, skip_reason | None)`. Whitelists only
    the request input fields, validates via the SAME `SportsAnalysisRequest` model the live endpoint uses,
    derives the regime, and re-serializes input-only.
  - `export_mlb_requests(raw_records, source)` -> `ExportResult` (exported records, `exported_count`,
    `skipped_count`, `skip_reasons`, `regime_distribution`, `source`).
  - `write_requests_file(records, path)` -> jsonl (the exact `--requests` format).
  - `read_raw_records(path)` -> raw records from a json array or jsonl file (utf-8-sig, BOM-tolerant); no
    network/db.
- `scripts/export_mlb_soak_requests.py` -- runnable: `--source`, `--out`, `--source-label`; prints the summary;
  exit 0 iff at least one record exported.

## what it is not

- NOT a retriever (no network/db/model call; the caller supplies raw records).
- NOT a credential reader (harvests nothing).
- NOT live-data evidence (no real export was run this slice).
- NOT routing, settlement, calibration, or buyer output.

## approved uses

- Normalize raw analyzer records from an approved source into a clean `--requests` file, then run the soak.

## disallowed uses

- Do not present synthetic/representative exports as live data.
- Do not commit generated request files or raw provider payloads (write `--out` to a scratch / out-of-repo
  path).
- Do not read DB credentials or scrape secret locations to obtain a source.

## workflow impact

Export -> soak is the two-step operator pipeline that unblocks the live-data evidence gate. Until a real export
+ clean live soak exists, registry-authoritative routing stays gated.

## truth hierarchy

The export is a pre-processing layer, never authority. Source, tests, and the live prompt remain authoritative.
A clean export proves only that records are well-formed input, not that any decision is correct.

## source or vault references to verify

- Export logic + input-only guarantee: `services/agent-service/app/services/mlb_soak_export.py`
  (`_INPUT_FIELDS` whitelist; `model_dump(exclude_none=True, include=set(_INPUT_FIELDS))`).
- Request contract: `services/agent-service/app/models/sports.py` `SportsAnalysisRequest`.
- Regime derivation reused: `services/agent-service/app/services/migration_readiness.py` `mlb_route_and_slots`.
- Soak consumer (round-trip): `services/agent-service/app/services/shadow_cohort_soak.py`
  `load_cohort_from_requests`.
- Tests: `services/agent-service/tests/test_mlb_soak_export.py` (12 cases, incl. an end-to-end export-script ->
  soak-script chain; all offline).

## example usage

From `services/agent-service`, when an approved raw source file exists (produced out-of-band, OUT OF REPO):

```
# 1. export raw records -> clean input-only --requests file (no model call, no network):
python scripts/export_mlb_soak_requests.py --source <scratch>/raw.jsonl --out <scratch>/reqs.jsonl \
    --source-label "persisted runs <date>"
#    summary json: exported_count / skipped_count / skip_reasons / regime_distribution / source / generated_path

# 2. run the live-data shadow soak on the exported file:
python scripts/run_shadow_cohort_soak.py --requests <scratch>/reqs.jsonl --sink <scratch>/soak.jsonl
#    exit 0 = GO (clean and whole cohort validated); inspect counts + regimes + unrepresented.
```

Raw source options (priority order; all out-of-band, no secrets harvested by the tool):
1. a local/dev retrieval path that emits the composed `SportsAnalysisRequest` inputs (none exists model-free
   today -- see deferred);
2. captured request-input artifacts (none exist today);
3. a sanctioned persisted-run export via approved application config (operator-run; not by this tool);
4. an operator-provided request-input json/jsonl file.

## agent prompt guidance

When asked to run the live-data soak: re-check access; if a raw source is reachable through an approved path,
export then soak; otherwise ship/keep the tool and record the blocker -- never fabricate live data, never
harvest credentials.

## acceptance criteria

- Valid records export into the `--requests` format; the exported file round-trips through
  `load_cohort_from_requests` and the soak goes GO (verified, offline).
- Invalid/incomplete records are skipped with a stable reason (`missing_identity` / `non_mlb` /
  `invalid_shape`); counts satisfy `exported + skipped == raw`.
- Any model output / artifact field in a raw record is stripped (never reaches the request input) -- verified.
- `regime_distribution` matches the regimes the soak computes (same `mlb_route_and_slots`).
- No model call, no network/db, no committed generated artifacts.

## validation rules (encoded)

- Identity: `homeTeam`/`awayTeam`/`gameDate` present and non-blank (null/empty/whitespace -> `missing_identity`).
- Competition: must be `mlb` (else `non_mlb`).
- Starter/market shape: validated by the nested pydantic models (bad shape -> `invalid_shape`).
- Regime derivable via `mlb_route_and_slots` (else `invalid_shape`).
- Output-stripping: only `_INPUT_FIELDS` are carried; nested `extra='ignore'` drops stray nested keys too.

## risks and failure modes

- **Live access still blocked.** No real export was produced; the live-data evidence gate remains open.
- **Source quality unknown.** The tool validates shape, not truthfulness -- a real export still depends on the
  source being a faithful capture of analyzer inputs.
- **Pretty-printed single-object source files are out of contract** (array or jsonl only); they raise rather
  than parse. Documented, not handled.
- **Regime-derivation reuse.** Export and soak share `mlb_route_and_slots`; if that changes, both move together
  (intended).

## deferred decisions

- A model-free `.NET` retrieval/preview endpoint that emits composed `SportsAnalysisRequest` inputs (none today;
  the analyze endpoint always calls the model; `GET {id}/artifact` returns the OUTPUT artifact only).
- A sanctioned persisted-run export path (needs approved DB config; out of scope for this tool).
- DB persistence of provenance, `.NET sourceDepth -> dataRegime` ownership, partial-evidence overlays --
  unchanged from prior slices.

## live export result (this slice)

- **Live access available?** NO. platform-api not running (no :5000/:5028/:7028/:8080); no model-free retrieval
  endpoint in `AgentRunsController` (analyze calls the model; `GET {id}/artifact` is output-only); no captured
  request-input artifacts on disk (`grep mlbStarterContext|baseballMarketContext *.json/*.jsonl` = 0); devcore-sql
  is up but extracting inputs would require harvesting DB credentials (prohibited; a credential scan was correctly
  blocked by the sandbox). No operator-provided file.
- **Export source used:** none (no live export run). Tool verified offline with representative raw SHAPES (clearly
  labeled, not live) + an end-to-end export-script -> soak-script chain.
- **Exported count / regime distribution:** n/a for live data (not run).
- **Why live behavior remains unchanged:** `sports_analyzer.py` and `.NET` unchanged this slice (git-confirmed);
  only the export module/script/tests + a BOM-tolerance fix to the soak loader changed (additive). No prompt /
  model / template / manifest change; validator stays default-off.
- **Tests run:** `pytest tests/test_mlb_soak_export.py -q` -> 12 passed; `pytest -q` -> **307 passed, 0 failed**;
  `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0; export-script demo (synthetic)
  -> 4 raw, 2 exported, 2 skipped, input-only.

## related docs

- [[live-data-shadow-soak-v1]] (the consumer + the access blocker), [[shadow-parallel-cohort-soak-v1]],
  [[live-prompt-routing-shadow-parallel-v1]], [[live-migration-readiness-v1]].

## recommended next slice

**Execute MLB Request Input Export + Live-Data Soak (when access exists):** obtain a raw source through an
approved path (bring up platform-api to capture composed requests, or export persisted run inputs via approved
config, or an operator-provided file), run
`python scripts/export_mlb_soak_requests.py --source <raw> --out <scratch>/reqs.jsonl` then
`python scripts/run_shadow_cohort_soak.py --requests <scratch>/reqs.jsonl --sink <scratch>/soak.jsonl`. If
`partial_evidence_unrepresentable > 0`, scope a partial-evidence overlay before routing; any
mismatch/assembly/sink/error = do not route. Only a clean live GO unlocks Registry-Authoritative Prompt v1.
