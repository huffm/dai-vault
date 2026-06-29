# Registry Canary Real Confirmation v1 -- Evidence Report

**status:** active doctrine (confirmation PASSED on one real allowlisted execution; canary returned to default-off)
**date:** 2026-06-28

## purpose

Record the result of exactly one operator-approved, paid real MLB execution that confirms the default-off
registry-authoritative prompt canary ([[registry-authoritative-prompt-canary-v1]]) works end-to-end: for an
allowlisted regime the registry prompt was selected as model input only after byte equality with the live
prompt, with no change to model behavior, one model call, and a normal artifact.

## problem it solves

The canary's byte-identity invariant was proven offline (tests + adversarial review) but never exercised on a
real request. This slice closes that gap with a single controlled real run, so migration readiness rests on
runtime evidence, not just static analysis.

## strategic fit

The runtime checkpoint between "the canary is correct on paper" and "the canary may be used on real allowlisted
traffic." It does not widen the allowlist or move toward broad migration.

## what it is

A single real execution + evidence capture. NO runtime code changed (no defect found). The agent-service was
restarted from the working tree with the canary enabled (default allowlist) and request capture enabled (for
evidence), one real MLB run was triggered via the platform, evidence was captured, and the service was returned
to default-off.

## what it is not

- NOT a code change (canary unchanged). NOT a broadening of the allowlist. NOT broad routing. NOT a second model
  call (exactly one). NOT a buyer change.

## the run

- Matchup: Los Angeles Dodgers @ San Diego Padres, gameDate 2026-06-28 (real slate).
- Trigger: `POST /api/agent-runs` (the normal live pipeline: platform retrieval -> capture+canary-enabled
  agent-service -> one model call).
- agentRunId: `21de423e-f36b-1410-816d-00373db4b724`.

## evidence (all success criteria met)

| criterion | result |
|---|---|
| HTTP status | **200** |
| artifact | status `completed`, normal (lean "Slight lean toward Padres...", confidence 0.675, posture monitor, 4 factors) |
| canary enabled | **true** (process started with `DAI_MLB_REGISTRY_PROMPT_CANARY=1`; `load_registry_prompt_canary_config` -> enabled, default 2-regime allowlist) |
| derived regime | `starter_enriched_market_missing` -- **allowlisted** |
| registry prompt selected | **true** (`select_model_prompt` source = `registry`) |
| model input byte-identical to live | **true** (`model_msg == build_mlb_user_message(...)`) |
| mismatch | **none** (0 "registry prompt canary MISMATCH" log lines; a mismatch would have logged and fallen back) |
| exactly one model call | **true** (1 "sports model-call cost" log line; status `ok`, finish_reason `stop` -- no truncation/retry) |
| second call / retry | **none** |

## how "registry selected + byte-identical" was established

On a canary SUCCESS the code emits no log (by design -- it silently uses the proven-identical bytes). The
runtime selection was therefore confirmed by three independent facts:
1. the agent-service process had the canary enabled (env-confirmed at startup);
2. `select_model_prompt` -- a pure function -- was reproduced on the EXACT analyzer input the live process
   received (captured via request capture to a scratch jsonl), with the same code, and returned
   `source="registry"` with `model_msg == live_prompt` byte-for-byte (assembledHash `6c4e9429c1dba4b5...`);
3. the agent-service log shows zero MISMATCH lines (equality failure would have logged loudly and fallen back
   to live).
Together these establish that the live request used the registry prompt as model input, byte-identical to the
live prompt. (A direct runtime "selected" log does not exist by design; adding one would be a code change and
no defect was found, so none was made.)

## why live behavior remained unchanged

The model input was byte-identical to the live `build_mlb_user_message` output, so the model received exactly
the same prompt it would have without the canary. One model call, normal artifact, HTTP 200. No
prompt/template/manifest/.NET change. `sports_analyzer.py` unchanged this slice.

## exact commands

```
# agent-service restarted from working tree with: DAI_MLB_REGISTRY_PROMPT_CANARY=1, DAI_MLB_REQUEST_CAPTURE=1,
#   DAI_MLB_REQUEST_CAPTURE_PATH=<scratch>/canary_capture.jsonl
curl -X POST http://localhost:5007/api/agent-runs -H "Content-Type: application/json" \
  -d '{"runType":"sports.matchup.analysis","agentProfileId":null,
       "input":{"competition":"mlb","homeTeam":"San Diego Padres","awayTeam":"Los Angeles Dodgers","gameDate":"2026-06-28"}}'
# evidence: derive regime from the captured input + reproduce select_model_prompt; grep the agent-service log.
# agent-service then restarted to DEFAULT (canary off, capture off).
```

## exact tests run

- `pytest tests/test_registry_prompt_canary.py -q` -> **10 passed**.
- `pytest -q` (full agent-service suite) -> **330 passed, 0 failed**.
- `python scripts/check_prompt_manifest.py` -> **OK** (8 templates, 9 recipes), exit 0.
- live: 1 real paid agent-run (HTTP 200).

## paid calls

Exactly **1** (the single confirmation run). No retries, no second call.

## acceptance criteria

Exactly one paid call; canary enabled for an allowlisted regime; registry prompt selected only after byte
equality; model input byte-identical to live; no mismatch; HTTP 200; artifact normal; tests + manifest pass;
evidence recorded; no push. ALL MET.

## risks and failure modes

- **Single regime confirmed at runtime.** The run landed in `starter_enriched_market_missing` only; the other
  allowlisted regime (`starter_enriched_market_backed_depth`) is proven by the prior real-cohort soak + offline
  tests but was not the one exercised in this single confirmation run.
- **Success has no direct runtime log.** Confirmation relies on the deterministic reproduction + the
  no-mismatch log. A future slice could add an opt-in info-level canary-selection log if richer live
  observability is wanted (a code change; not done here).
- **No code change.** No defect was found, so the canary is unchanged.

## deferred decisions

- A second confirmation run landing in `starter_enriched_market_backed_depth` (optional; needs paid approval).
- Optional canary-success observability log. Expanding the allowlist to the other 7 regimes (needs real-soak
  evidence). Broad registry-authoritative routing. DB persistence; `.NET sourceDepth -> dataRegime` ownership;
  caching -- unchanged deferrals.

## related docs

- [[registry-authoritative-prompt-canary-v1]] (the canary it confirms), [[real-cohort-live-soak-v1]] (allowlist
  evidence), [[mlb-analyzer-request-capture-v1]].

## recommended next slice

**Multi-Slate Regime Coverage v1 (operator-approved):** earn real-soak evidence for the other 7 regimes across
additional slates/markets before any allowlist widening. Optionally first run a single confirmation that lands
in `starter_enriched_market_backed_depth`. Only a complete, clean allowlist moves toward broad
Registry-Authoritative Prompt migration.
