# Artifact Cost Guardrails v1

**date:** 2026-06-09
**status:** implemented. narrow FastAPI instrumentation + tests in `dai`; report/handoff/ledger in `dai-vault`. no prompt, buyer-copy, confidence/posture/lean/signal/decision, schema, source, probe-refresh, pricing/Stripe, tenant/auth/dashboard/deployment change.
**scope:** instrument and bound the single sports artifact model-call boundary enough to support unit-cost discipline.

## Purpose

Treat the model call as cost of goods sold for one manufactured artifact. The model-call inventory established there is exactly one model call per sports run (`FastAPI POST /api/sports/analyze -> sports_analyzer._call_model`, `gpt-4o-mini`), and that `response.usage`, cost, latency, and request metadata were not captured. This slice measures and bounds the cost of producing one unit. It is cost telemetry, not pricing and not billing.

Primary question: for each sports artifact run, can the system capture and expose enough model-call metadata to estimate manufacturing cost and prevent uncontrolled model spend? Answer: yes, for the single centralized call, log-only in v1.

## Scope

- instrument the one sports model-call boundary with token usage, cost estimate, latency, status, finish reason, and request id.
- add a small, deterministic, unit-tested cost estimator with a clearly labelled configured pricing table.
- add a conservative `max_completion_tokens` cap and a per-call timeout; document the retry posture.
- emit a structured internal cost record; do not change buyer/artifact semantics or contracts.

## Non-goals

- no pricing, no Stripe, no tenant billing, no dashboards, no broad metering platform.
- no prompt/buyer-copy/confidence/posture/lean/signal/decision change.
- no evidence-sufficiency band gate (that is Confidence Calibration Rules' future implementation slice).
- no source/probe-refresh/schema/artifact-contract change; no auth/tenant/Kubernetes/deployment change.
- no push.

## Model-call boundary instrumented

`dai/services/agent-service/app/services/sports_analyzer.py :: _call_model` -- the single `client.chat.completions.create(...)` call for sports analysis. The call now:

- runs inside a latency timer (`time.perf_counter`).
- on success and on failure, builds a structured cost record and logs it (failure record then re-raises the original error unchanged -- model failures are not swallowed).
- the return value and the parsed `SportsAnalysisResponse` are byte-for-byte unchanged. No new field is added to the analyzer response or the .NET wire contract, deliberately, to avoid touching the canonical wire shape.

A new pure module `dai/services/agent-service/app/services/model_metering.py` owns the cost math and metadata shape (no i/o, no model calls), so it is unit-testable in isolation.

## Metadata captured

One structured JSON record per run, emitted to the dedicated `devcore.cost` logger (separate from buyer/analyzer logs):

- `model` (e.g. `gpt-4o-mini`)
- `inputTokens`, `outputTokens`, `totalTokens` (from `response.usage`; null when usage absent)
- `usageAvailable` (true/false)
- `latencyMs`
- `status` (`ok` or `error:<ExceptionType>`)
- `finishReason` (e.g. `stop`)
- `requestId` (best-effort from the SDK response, e.g. `req_...`)
- `estimatedInputCost`, `estimatedOutputCost`, `estimatedTotalCost`, `currency` (USD)
- `pricingSource` (configured-estimate label), `costEstimateVersion` (`v1`)
- `competition`

Observed live record (full-chain MLB smoke, 2026-06-09): `inputTokens 2622, outputTokens 362, totalTokens 2984, usageAvailable true, latencyMs 8185, status ok, finishReason stop, requestId req_..., estimatedTotalCost 0.0006105 USD`. So one sports artifact currently costs roughly 0.06 US cents to manufacture at the model boundary.

## Pricing estimate approach

A small hand-maintained `PRICING` table in `model_metering.py`: USD per 1,000,000 tokens per model. `gpt-4o-mini` configured at input 0.15 / 1M and output 0.60 / 1M (public list pricing). `estimate_cost` is deterministic, rounds to fixed precision, and returns null costs (fail safe) when tokens are missing or the model has no configured price. Every record carries `pricingSource = "openai-public-list-configured-estimate (maintain manually; not billing truth)"` and `costEstimateVersion = "v1"` so a reader knows it is an estimate, not a billed amount. Stripe remains revenue truth; this is internal manufacturing-cost telemetry only. Pricing must be maintained by hand when provider prices change (recorded in ledger entry 22).

## Max token decision

Added: `max_completion_tokens = 1500` on the sports call. It caps a pathological runaway generation while leaving generous headroom -- the live smoke produced only 362 completion tokens with `finishReason = "stop"`, confirming no truncation of the JSON artifact. The cap is a safety bound, not an output shaper; it does not change normal output. If a future prompt makes the artifact larger and `finishReason` starts reporting `length`, raise the cap (the cost record surfaces this).

## Timeout decision

Added: a 30s per-call `timeout` on the sports call. The .NET `FastApiClient` HttpClient timeout is 90s, so a stuck model call now fails at the FastAPI boundary with a recorded `error:` cost record (and 30s latency) rather than hanging toward the upstream limit. Narrow and local; no broad timeout architecture introduced.

## Retry posture

No custom retry added in v1. The OpenAI SDK default applies and is documented at the call site: at most 2 bounded retries on transient errors (429/5xx/timeout) with backoff, so one logical run is at most 3 attempts. Left unchanged on purpose (bounded, predictable spend). The final outcome is captured in the cost record `status`. No unbounded or custom retry was introduced.

## Fail-safe behavior

Cost telemetry can never break artifact generation. `_log_call_metadata` wraps extraction in try/except and logs a warning on any failure. Missing `response.usage` records `usageAvailable false` with null tokens/costs rather than raising. A failed model call still records a structured failure record (`status: error:<Type>`) and then re-raises the original exception so upstream behavior is unchanged.

## Smoke result

Services: Docker/`devcore-sql` up, FastAPI `/api/ping` 200, .NET `/health` 200.

Full chain `POST /api/agent-runs` (MLB Orioles vs Mariners, 2026-06-12): HTTP 200, `status completed`, `durationMs 9070`. Artifact intact: `artifactVersion sports_decision_artifact_v3`, `cognitiveProtocol` present, buyer projection (`lean` + `summary`) present. Buyer copy unchanged. The cost record was emitted with full token/cost/latency data and `finishReason stop` (see Metadata captured).

## Tests / checks run

- TDD for `model_metering.py`: wrote `tests/test_model_metering.py` first, watched it fail (module missing), implemented to green. 6 tests: known-model cost math, missing-tokens unavailable, unknown-model unavailable, full metadata with usage, safe metadata without usage, failure-status with no usage.
- Full agent-service suite: 111 passed (6 new + 105 existing), no regressions.
- Import check on `sports_analyzer` after instrumentation: clean.
- Full-chain smoke: artifact generated, v3 + cognitiveProtocol + buyer projection intact, cost record captured.
- Repo verification: `git diff --check`, added-line exact-path scan, vault non-ASCII added-line scan.

## Remaining gaps

- log-only: no persistent sink, no aggregation, no per-tenant cost attribution, no spend cap / budget enforcement. The system measures unit cost; it does not yet bound total spend across many runs (only per-call token/timeout bounds exist). Recorded as ledger entry 22.
- pricing is hand-maintained and must be updated when provider prices change.
- request id is best-effort from the SDK response object; if a future SDK changes that accessor, `requestId` becomes null (fail safe).
- the separate non-sports chat/assist/gRPC model call (`main.py`) is intentionally not instrumented in this slice -- it is not part of sports artifact manufacturing.

## Recommended next slice

- Confidence Calibration Rules v1 implementation (band-gating from the doctrine already specced) remains the strongest product-quality follow-up.
- Model Cost Sink and Budget v1 (persist + aggregate cost records, optional spend cap), sequenced with the tenant/economic boundary, is the natural continuation of this slice when per-tenant accounting or a spend ceiling is actually needed.

## What was not changed

- no FastAPI prompt, parser, or analyzer semantics (only the call boundary + a new pure metering module)
- no .NET, Angular, schema, or wire-contract change
- no confidence/posture/lean/signal/decision values
- no buyer-facing artifact field or buyer UI
- no pricing/Stripe/tenant-billing/dashboard/auth/deployment work
- no source/probe-refresh change
- no `OPENAI_API_KEY` or secret change
