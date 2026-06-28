# Real-Cohort Live Soak v1 -- Evidence Report

**status:** active doctrine (real multi-regime live soak RAN; clean GO on 15 real games / 2 regimes)
**date:** 2026-06-28

## purpose

Record the result of running an operator-approved real MLB cohort through the live analyzer / request-capture /
shadow-soak path, as real-data evidence for prompt-registry migration readiness. This is NOT routing: the live
prompt stayed authoritative; the registry prompt was only compared, never used as model input.

## problem it solves

Prior soaks proved shadow==live on representative fixtures and on a single real game (n=1). Migration readiness
needs real-data evidence across more than one regime. This slice produced that: a clean GO across the full real
MLB slate, spanning two real regimes.

## strategic fit

The evidence gate before any Registry-Authoritative Prompt v1 slice. A clean real cohort unlocks routing
consideration ONLY for the regimes the real cohort actually proved clean.

## mental model

Capture (analyzer boundary) -> scratch jsonl -> shadow soak -> GO/NO-GO + regime distribution. Real games make
the cohort; the soak measures whether the registry prompt reproduces the live prompt byte-for-byte on them.

## what it is

An operational run, not new code (no runtime code changed this slice). It used the already-shipped pipeline:
- `app/services/mlb_request_capture.py` (default-off capture, enabled for this run via env),
- the platform-api retrieval path (real starter/market from statsapi + odds),
- `scripts/run_shadow_cohort_soak.py --requests` (the live-data soak).

## what it is not

- NOT registry-authoritative routing, NOT settlement/scoring, NOT outcome claims, NOT a second model call per
  run, NOT a buyer change. No prompt/template/manifest/.NET change.

## approval path used

PAID cohort path (path 2), explicitly operator-approved this slice ("Run full slate"). The preferred
no-new-model-call path (path 1) was investigated and rejected as insufficient: persisted `InputJson` (via the
sanctioned `GET /api/agent-runs/{id}`) is identity-only (`Competition/HomeTeam/AwayTeam/GameDate`) -- it does
NOT contain the retrieved starter/market contexts, and there is no model-free retrieval endpoint, so path 1
would collapse every game to `starter_missing_market_missing` (non-representative). Full inputs require a real
analyzer run, hence the approved paid cohort.

## whether paid model calls were run

YES -- one model call per game (the normal live analyze call; no SECOND call per run). 14 new runs this slice
(+ 1 from the prior slice) = 15 real games. Each was a real `POST /api/agent-runs` (HTTP 200, real artifact).

## cohort source and size

Source: the real MLB slate for 2026-06-28 (15 games, from statsapi.mlb.com), each run through the platform-api
retrieval + the capture-enabled agent-service. Size: 15. Capture path: scratch `live_capture.jsonl` (out of
repo). Sink path: scratch `cohort_soak.jsonl` (out of repo). NEITHER committed.

## regime distribution

- `starter_enriched_market_backed_depth`: 2
- `starter_enriched_market_missing`: 13

Real data observation: today's slate is enriched-starter-heavy (announced probable pitchers with season stats),
and the odds API returned multi-book depth for only 2 games (the rest had no market at retrieve time -> market
missing). So the real cohort exercised 2 of the 9 regimes; the other 7 remain proven only on representative
fixtures (a real-data coverage gap, not a failure).

## summary counts (real cohort)

```
cohort_size:                       15
attempted:                         15
matched:                           15
captured:                          15
mismatched:                         0
assembly_failed:                    0
sink_failed:                        0
errored:                            0
skipped_no_sink:                    0
skipped_disabled:                   0
partial_evidence_unrepresentable:   0
persisted_provenance_lines:        15
clean:  true
go:     true
exit code: 0
```

Provenance sanity: 15 lines, all `mode=shadow`, all `lifecycle=shadow_only`, 15 distinct 64-char assembledHash,
2 distinct regimes, 15 distinct matchups. Captures verified input-only (no model output keys in any record).

## mismatches / failures / partial-evidence details

NONE. Zero mismatch / assembly_failed / sink_failed / errored / partial_evidence_unrepresentable on all 15 real
games. Shadow prompt == live prompt byte-for-byte across the cohort.

## exact commands

```
# (services up: devcore-sql docker; platform-api :5007 Development; agent-service :8000 capture-enabled)
# trigger each real game (one paid analyze call each):
curl -X POST http://localhost:5007/api/agent-runs -H "Content-Type: application/json" \
  -d '{"runType":"sports.matchup.analysis","agentProfileId":null,
       "input":{"competition":"mlb","homeTeam":"<home>","awayTeam":"<away>","gameDate":"2026-06-28"}}'
# soak the captured real cohort:
python scripts/run_shadow_cohort_soak.py --requests <scratch>/live_capture.jsonl --sink <scratch>/cohort_soak.jsonl
```

## exact tests run

- `pytest -q` (full agent-service suite) -> **320 passed, 0 failed** (no code changed this slice).
- `pytest tests/test_mlb_request_capture.py tests/test_shadow_cohort_soak.py tests/test_shadow_validation.py -q`
  -> 44 passed.
- `pytest tests/test_mlb_prompt_equivalence.py tests/test_mlb_soak_export.py -q` -> 31 passed.
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- live: 14 real agent-runs (HTTP 200) + 1 real-cohort soak (GO).

## why live behavior remains unchanged

No runtime code changed this slice. Each game ran the normal live analyzer (live prompt authoritative, one model
call, normal artifact returned HTTP 200); request capture is an input-only side-effect that runs before the
model call and does not alter `user_msg` or the response. `.NET` unchanged.

## acceptance criteria

- More than one regime exercised (2 of 9). MET (threshold: >1 if available).
- Clean GO across the real cohort. MET (15/15, zero failures).
- Routing remains gated; only proven-clean regimes are candidates for a later routing slice.

## risks and failure modes

- **Regime coverage is partial.** Only 2 enriched-starter regimes appeared in today's real slate; 7 regimes
  (named/missing starter; backed market) are unproven on real data. A future slice should capture cohorts on
  days/markets that exercise those regimes before routing them.
- **Single-writer capture.** The 15 sequential runs used one append-only file safely (sequential, no
  concurrency). High-concurrency capture remains out of scope.
- **n still modest.** 15 games / 1 slate. More slates strengthen the evidence.

## deferred decisions

- Registry-authoritative routing (still gated -- not unlocked by one clean slate).
- Coverage of the other 7 regimes on real data.
- DB persistence of provenance; `.NET sourceDepth -> dataRegime` ownership; partial-evidence overlays.

## related docs

- [[mlb-analyzer-request-capture-v1]] (the capture), [[live-data-shadow-soak-v1]] (the soak harness + access
  history), [[mlb-request-input-export-v1]], [[shadow-parallel-cohort-soak-v1]].

## recommended next slice

**Multi-Slate Regime Coverage v1 (operator-approved):** capture real cohorts across additional slates/markets
to exercise the named/missing-starter and backed (single-book) market regimes that today's slate did not, until
each of the 9 regimes has a clean real soak. Only then consider **Registry-Authoritative Prompt v1**, and only
for regimes with a clean real soak. Alternatively, persist retrieved starter/market context at retrieve time
(a small .NET change) so future cohorts can be rebuilt from persisted data without new model calls -- evaluate
that as its own slice.
