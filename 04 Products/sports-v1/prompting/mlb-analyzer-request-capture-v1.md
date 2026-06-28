# MLB Analyzer Request Capture v1

**status:** active doctrine (capture shipped + verified; first LIVE capture + live-data soak RAN, GO on n=1)
**date:** 2026-06-28

## purpose

Provide a default-off way to capture the real input-only MLB analyzer request context at the analyzer boundary,
into a scratch JSONL that feeds the live-data shadow soak -- without changing the live prompt, model call,
buyer output, confidence, or artifact behavior. This slice shipped the capture and, with the operator bringing
the dev services up, RAN the first live capture + live-data soak on real data.

## problem it solves

The live-data soak ([[live-data-shadow-soak-v1]]) and the export tool ([[mlb-request-input-export-v1]]) were
blocked on one thing: a real source of MLB analyzer inputs. Capture closes the loop by recording exactly what
the analyzer received as input on a real run, with no model output mixed in, directly soak-consumable.

## strategic fit

Final input-capture step of the prompting-migration track. Capture (this) -> live-data soak -> (gated)
registry-authoritative routing. The live soak that this enabled is the evidence gate before routing.

## mental model

Three separate concepts, kept distinct:
- **Request capture** (this) records what the analyzer RECEIVED as input.
- **Shadow validation** checks whether the registry prompt equals the live prompt.
- **Provenance capture** records the shadow prompt assembly lineage.
Capture is independent of the other two -- it does not require shadow validation to be enabled.

## what it is

- `app/services/mlb_request_capture.py`:
  - `MlbRequestCaptureConfig(enabled=False, path=None)` + `load_mlb_request_capture_config(env)` reading
    `DAI_MLB_REQUEST_CAPTURE` (truthy) and `DAI_MLB_REQUEST_CAPTURE_PATH`. Default DISABLED.
  - `mlb_request_input_record(...)` -- pure; builds the input-only record (the `SportsAnalysisRequest` shape)
    from the typed contexts; absent contexts omitted; no model output exists at capture time (it runs before
    the model call).
  - `capture_mlb_request(config, ...)` -- appends one record to the scratch jsonl; no-op (False) when disabled
    or no path; a write failure is logged and returned False, never raised.
- `analyze_mlb` gains a keyword-only `capture_config`; the capture block runs after `build_mlb_user_message`,
  before the model call, wrapped in a guard so NOTHING (even a broken import) can break the live request.

## what it is not

- NOT routing, NOT a second model call, NOT a second artifact, NOT a buyer change, NOT settlement/scoring.
- NOT a capture of model output / artifact / confidence / buyer copy (input-only by construction).

## approved uses

- Enable on a controlled/canary slice to capture real analyzer inputs to a scratch path, then run the soak.

## disallowed uses

- Do not enable by default or on high-concurrency production (single-writer file; see risks).
- Do not commit captured JSONL (scratch / out-of-repo only).
- Do not treat the captured input as evidence of decision quality.

## workflow impact

Capture-enabled analyzer run -> scratch jsonl -> `run_shadow_cohort_soak.py --requests` -> GO/NO-GO. This is
the live evidence pipeline; capture output is directly soak-consumable (no export step needed for clean
captures).

## truth hierarchy

Capture is an observability side-effect, never authority. The live prompt and model call remain authoritative
and unchanged; capture only records inputs.

## source or vault references to verify

- Capture module + input-only guarantee: `services/agent-service/app/services/mlb_request_capture.py`.
- Analyzer wiring (default-off, guarded, before model call): `services/agent-service/app/services/sports_analyzer.py`
  `analyze_mlb`.
- Request shape: `services/agent-service/app/models/sports.py` `SportsAnalysisRequest`.
- Soak consumer: `services/agent-service/app/services/shadow_cohort_soak.py` `load_cohort_from_requests`.
- Tests: `services/agent-service/tests/test_mlb_request_capture.py` (13 cases).

## enablement flags

- `DAI_MLB_REQUEST_CAPTURE` = `1|true|yes|on` (default off).
- `DAI_MLB_REQUEST_CAPTURE_PATH` = scratch jsonl path (capture is a no-op without it).

## captured record shape (one jsonl line)

`{competition:"mlb", homeTeam, awayTeam, gameDate, mlbStarterContext?, baseballMarketContext?}` -- the
`SportsAnalysisRequest` input shape. Absent contexts are omitted.

## excluded fields

`lean`, `summary`, `confidence`, `posture`, `counterCase`, `watchFor`, `factors`, `groundedSignals`,
artifact/quality fields, secrets, credentials, raw unnecessary provider payloads. (Verified: a real captured
record contained exactly the 6 input keys and zero output keys.)

## example usage

Enable capture on the agent-service, drive a real run via the platform, soak the capture:

```
# agent-service started with: DAI_MLB_REQUEST_CAPTURE=1 DAI_MLB_REQUEST_CAPTURE_PATH=<scratch>/cap.jsonl
# platform-api (ASPNETCORE_ENVIRONMENT=Development) up; devcore-sql up.
# trigger a real mlb run (platform retrieves real starter/market and posts to the capture-enabled agent-service):
curl -X POST http://localhost:5007/api/agent-runs -H "Content-Type: application/json" \
  -d '{"runType":"sports.matchup.analysis","agentProfileId":null,
       "input":{"competition":"mlb","homeTeam":"Boston Red Sox","awayTeam":"New York Yankees","gameDate":"2026-06-28"}}'
# soak the captured real input:
python scripts/run_shadow_cohort_soak.py --requests <scratch>/cap.jsonl --sink <scratch>/live_soak.jsonl
```

## live capture + soak result (this slice)

- **Live services available?** YES (operator brought them up). devcore-sql (docker) up; platform-api on :5007
  (Development); agent-service on :8000 with capture enabled (scratch path).
- **Capture run?** YES. One controlled real MLB run (Yankees @ Red Sox, 2026-06-28) -> HTTP 200, real model
  output (Sonny Gray vs Carlos Rodon; market support for Boston). Capture wrote ONE real input-only record:
  real starters with season stats (Gray ERA 2.95 / Rodon ERA 3.70), real 9-book DraftKings market (consensus
  home, median home implied prob 0.528, disagreement 0.020, median total 8.0). Regime
  `starter_enriched_market_backed_depth`. (A larger cohort was intentionally NOT fired -- repeated paid model
  calls need explicit operator approval; one controlled invocation is the slice minimum.)
- **Soak run?** YES. `run_shadow_cohort_soak.py --requests <capture> --sink <scratch>` on the real record:
  cohort_size 1, attempted 1, matched 1, captured 1, 0 mismatch/assembly/sink/error,
  partial_evidence_unrepresentable 0, regimes {starter_enriched_market_backed_depth: 1}, clean true, GO,
  persisted_provenance_lines 1, exit 0. Provenance: mode=shadow, lifecycle=shadow_only, 64-char assembledHash,
  3 pieces. This is the first shadow-equivalence GO on REAL captured analyzer input (not fixtures).
- **Generated file policy.** Capture + soak files are scratch / out-of-repo; NONE committed.

## why live behavior remains unchanged

`build_mlb_user_message`, the model call, and the response are unchanged; capture runs before the model call,
reads (never mutates) the context objects, and its result is ignored by the request. Disabled (default) is
byte-identical (proven by test + adversarial review). The one live run returned a normal real artifact (HTTP
200) -- capture did not alter it. `.NET` unchanged this slice.

## acceptance criteria

- Disabled = byte-identical, no file, no import touch (verified: test + review).
- Enabled = one input-only jsonl record per run; output fields never present (verified on real data).
- Captured record round-trips through the soak loader and goes GO (verified on real data).
- Write failure observable + non-fatal (verified).

## risks and failure modes

- **Single-writer.** Concurrent appends under high-concurrency traffic could tear a jsonl line (breaks the soak
  loader, never the live request). Use on a controlled/canary slice only.
- **n=1 live evidence.** One real game is a clean GO but not sufficient for routing; a larger real cohort is
  needed (gated on operator approval for the paid calls).
- **Starter handedness "?" in real data.** statsapi did not return pitch hand for this game; capture recorded
  it faithfully and the prompt still matched (both live + shadow render the same "?"). Not a defect.

## deferred decisions

- Larger real-cohort live soak (needs operator approval for repeated model calls).
- Registry-authoritative routing (still gated -- not unlocked by n=1).
- DB persistence of provenance; `.NET sourceDepth -> dataRegime` ownership; partial-evidence overlays.

## related docs

- [[live-data-shadow-soak-v1]], [[mlb-request-input-export-v1]], [[shadow-parallel-cohort-soak-v1]],
  [[live-prompt-routing-shadow-parallel-v1]].

## recommended next slice

**Real-Cohort Live Soak v1 (operator-approved):** with services up and capture enabled, fire an
operator-approved batch of real MLB runs (the full real slate, ~15 games) to build a real multi-regime cohort,
soak it, and record the regime distribution + any partial_evidence_unrepresentable / mismatch. Only a clean GO
across a representative real cohort should move toward Registry-Authoritative Prompt v1. Alternatively, source
persisted `InputJson` from `/api/agent-runs` via the sanctioned API + the export tool (no new model calls).
