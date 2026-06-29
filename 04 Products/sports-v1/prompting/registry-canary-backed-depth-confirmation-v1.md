# Registry Canary Backed-Depth Confirmation v1 -- Evidence Report

**status:** active doctrine (confirmation PASSED for the second allowlisted regime; canary returned to default-off)
**date:** 2026-06-28

## purpose

Record one operator-approved paid real MLB execution confirming the default-off registry-authoritative prompt
canary ([[registry-authoritative-prompt-canary-v1]]) for the SECOND allowlisted regime,
`starter_enriched_market_backed_depth`. Together with [[registry-canary-real-confirmation-v1]]
(`starter_enriched_market_missing`), both allowlisted regimes now have real-execution confirmation.

## problem it solves

The prior confirmation run landed in `starter_enriched_market_missing` only; the backed-depth regime was proven
by the cohort soak + offline tests but not by a single targeted real run. This slice closes that gap.

## what it is

A single real execution + evidence capture. NO runtime code changed (no defect found). The agent-service was
restarted from the working tree with the canary enabled (default 2-regime allowlist) and request capture
enabled (for evidence), one real backed-depth MLB run was triggered, evidence was captured, and the service was
returned to default-off.

## what it is not

- NOT a code change. NOT an allowlist expansion. NOT broad routing. NOT a second model call (exactly one). NOT a
  buyer change.

## backed-depth candidate availability

YES. New York Yankees @ Boston Red Sox derived backed-depth in both the prior confirmation slice and the
real-cohort soak (a reliably multi-book marquee market), so it was the best-justified single candidate. The run
confirmed it: market present, bookCount 6, consensusSide home -> `starter_enriched_market_backed_depth`.

## the run

- Matchup: New York Yankees @ Boston Red Sox, gameDate 2026-06-28 (real slate).
- Trigger: `POST /api/agent-runs` (normal live pipeline; one model call).
- agentRunId: `27de423e-f36b-1410-816d-00373db4b724`.

## evidence (all success criteria met)

| criterion | result |
|---|---|
| HTTP status | **200** |
| artifact | status `completed`, normal (lean "Slight lean toward Boston Red Sox...", confidence 0.8, posture monitor) |
| canary enabled | **true** (process started with `DAI_MLB_REGISTRY_PROMPT_CANARY=1`, default allowlist) |
| market depth | present, **bookCount 6, consensusSide home** (multi-book consensus) |
| derived regime | **`starter_enriched_market_backed_depth`** -- the target, allowlisted |
| registry prompt selected | **true** (`select_model_prompt` source = `registry`) |
| model input byte-identical to live | **true** (`model_msg == build_mlb_user_message(...)`) |
| assembledHash | `5c4b7402cc1b0dcadaa2d44396491f0c2897b197ee58da0b5c97a33bad831126` |
| exactly one model call | **true** (1 "sports model-call cost" line; status `ok`, finish_reason `stop`) |
| retries / second call | **none** |
| mismatch | **none** (0 "registry prompt canary MISMATCH" lines) |

## how "registry selected + byte-identical" was established

Same method as the prior confirmation (a canary success emits no log by design): (1) the agent-service process
had the canary enabled (env-confirmed at startup); (2) `select_model_prompt` -- a pure function -- was
reproduced on the EXACT analyzer input the live process received (captured via request capture), returning
`source="registry"` with `model_msg == live_prompt` byte-for-byte and assembledHash above; (3) the agent-service
log shows zero MISMATCH lines. Together these establish the live request used the registry prompt as model
input, byte-identical to the live prompt.

## why live behavior remained unchanged

Model input byte-identical to the live `build_mlb_user_message` output; one model call; normal artifact; HTTP
200. No prompt/template/manifest/.NET change. `sports_analyzer.py` unchanged this slice.

## exact tests run

- `pytest tests/test_registry_prompt_canary.py -q` -> **10 passed**.
- `pytest -q` (full agent-service suite) -> **330 passed, 0 failed**.
- `python scripts/check_prompt_manifest.py` -> **OK** (8 templates, 9 recipes), exit 0.
- live: 1 real paid agent-run (HTTP 200).

## paid calls

Exactly **1** (the backed-depth confirmation run). No retries, no second call.

## service / env posture after slice

agent-service :8000 restarted to DEFAULT (canary OFF, capture OFF); platform-api :5007; devcore-sql up. Canary
capture/response/log files are scratch/out-of-repo; NONE committed.

## acceptance criteria

Backed-depth candidate available; exactly one paid call; canary enabled; derived regime
`starter_enriched_market_backed_depth`; registry prompt selected only after byte equality; model input
byte-identical; assembledHash recorded; one model call; zero retries; zero mismatch; HTTP 200; artifact normal;
tests + manifest pass; service returned to default; no push. ALL MET.

## remaining risks and gaps

- **Allowlist regimes are fully confirmed at runtime; the other 7 regimes are not.** Both allowlisted regimes
  now have a real-execution confirmation, but the remaining 7 regimes still have no real-soak evidence and stay
  off the allowlist.
- **Single run per regime.** Each allowlisted regime has one confirming real run; not a large sample.
- **No code change.** No defect found; the canary is unchanged.

## deferred decisions

- Multi-Slate Regime Coverage to earn real-soak evidence for the other 7 regimes. Optional canary-success
  observability log. Broad registry-authoritative routing. DB persistence; `.NET sourceDepth -> dataRegime`
  ownership; caching -- unchanged deferrals.

## related docs

- [[registry-authoritative-prompt-canary-v1]] (the canary), [[registry-canary-real-confirmation-v1]] (the first
  allowlisted regime), [[real-cohort-live-soak-v1]] (allowlist evidence).

## recommended next slice

**Multi-Slate Regime Coverage v1 (operator-approved):** with both allowlisted regimes now runtime-confirmed,
earn real-soak evidence for the other 7 regimes (named/missing starter; single-book backed market) across
additional slates/markets before any allowlist widening. Only a complete, clean allowlist moves toward broad
Registry-Authoritative Prompt migration.
