---
title: "Starter-Missing Registry Canary Confirmation v1"
type: "evidence-report"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Starter-Missing Registry Canary Confirmation v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - provenance
  - source-depth
related:
  - "06 Execution/reports/default-allowlist-widening-v1.md"
  - "06 Execution/reports/live-scheduled-starter-missing-soak-v1.md"
---

# Starter-Missing Registry Canary Confirmation v1 -- Evidence Report

**status:** active doctrine (both starter_missing regimes runtime-confirmed registry-authoritative; allowlist NOT widened)
**date:** 2026-06-29

## purpose

Runtime-confirm that the two real-soak-clean starter_missing regimes route registry-authoritative under explicit
operator-enabled canary conditions, without widening `DEFAULT_ALLOWLIST`. This is a controlled paid-canary
confirmation slice, not a promotion slice. It converts "these regimes route deterministically in tests" into
"these regimes route registry-authoritative on a real paid model call", which is the prerequisite evidence for a
later, separate allowlist-widening decision.

## start state

Phase 3.2 Global Prompt Routing Hardening v1 committed and clean before this slice: dai `3f1b21f` (main, synced),
dai-vault `088f039` (main, synced). The `PromptRouteDecision` contract, `decide_model_prompt`, the
`select_model_prompt` shim, and analyzer route-decision logging were all present. No Phase 3.2 work was mixed into
this slice.

## target regimes (exact code names)

- `starter_missing_market_missing`
- `starter_missing_market_backed_depth`

Both were already real-soak-clean (see [[starter-missing-regime-capture-v1]]), so the registry prompt is proven
byte-identical to the live prompt for these shapes; the canary therefore routes registry-authoritative for them
when explicitly enabled.

## default allowlist before / after

UNCHANGED. In code, `registry_prompt_canary.py :: DEFAULT_ALLOWLIST` remains exactly:

```
("starter_enriched_market_backed_depth", "starter_enriched_market_missing")
```

The two starter_missing regimes were enabled for this run ONLY via the env override, never by editing
`DEFAULT_ALLOWLIST`. The driver printed the live config at runtime to prove this:
`enabled=True regimes=('starter_missing_market_missing', 'starter_missing_market_backed_depth')` while
`DEFAULT_ALLOWLIST=('starter_enriched_market_backed_depth', 'starter_enriched_market_missing')`.

## operator canary config used

```
DAI_MLB_REGISTRY_PROMPT_CANARY=1
DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES=starter_missing_market_missing,starter_missing_market_backed_depth
```

`load_registry_prompt_canary_config()` parses the comma-separated `_REGIMES_ENV` as the operator's explicit
allowlist override; when absent it falls back to `DEFAULT_ALLOWLIST`. The override is the documented,
intentional mechanism for canary eligibility without code promotion.

## execution method (and live-retrieval blocker)

The full live stack was down at slice time: agent-service :8000 and platform-api :5007 unreachable, and the
Docker daemon (which hosts `devcore-sql`) not running. Real-game input retrieval (statsapi probable pitchers +
odds) flows through platform-api `POST /api/agent-runs`, which needs `devcore-sql`, so live-scheduled retrieval
was unavailable.

The paid model call and the `PromptRouteDecision` both live in agent-service (`analyze_mlb` -> registry canary ->
`_call_model`), so runtime confirmation does not require the platform layer. Per the slice's sanctioned fallback,
a fixture-based canary was used: `analyze_mlb` was driven directly with representative starter_missing inputs
(the same shapes the equivalence/migration suites prove byte-equal to live), under the operator canary env. This
is real runtime + a real paid model call; it is NOT a live-scheduled game. Game labels below are representative
identities reused from the prior capture slate, not confirmed live matchups.

Model: `gpt-4o-mini`, one `chat.completions.create` per run (the canary adds no second call). A spy on
`run_mlb_prompt_canary` captured the exact decision the analyzer used; a wrapper on `_call_model` counted the
real paid call.

## non-paid tests run (before paid approval)

From `services/agent-service` (venv python):

- `pytest tests/test_prompt_route_decision.py tests/test_registry_prompt_canary.py tests/test_prompt_provenance.py
  tests/test_prompt_registry_contract.py -q` -> **74 passed**.
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.

## paid calls run

**4** real `gpt-4o-mini` calls (one per run; no second call/retry beyond the SDK default), operator-approved
(4 runs, "you run it"). Approximate cost: well under $0.01 total (small JSON payloads on gpt-4o-mini).

Note: an initial driver attempt failed before any network call (dotenv resolved the scratchpad dir, not
agent-service, so `OPENAI_API_KEY` was unset and `_get_client` raised) -- **0 paid calls** on that attempt. The
fix (`load_dotenv(find_dotenv(usecwd=True))`) was applied and the 4 approved runs then executed cleanly.

## route decision evidence

All four runs: `status=ok`, `modelCallCount=1`, `promptSource=registry`, `registryAuthoritativeEnabled=true`,
`legacyFallbackUsed=false`, `regimeAllowlisted=true`, `fallbackReason=null`. Buyer artifact keys identical across
all four (`confidence, counter_case, factors, lean, lean_side, posture, protocol, signals_used, summary,
watch_for, what_would_change_the_read`).

| runId | game (representative) | selectedDataRegime | selectedPromptRecipeId | ver | assembledHash (prefix) |
|---|---|---|---|---|---|
| 0d8ca87ea9e7 | Arizona Diamondbacks @ San Francisco Giants | starter_missing_market_missing | mlb.pregame.analysis.starter_missing_market_missing.v1 | v1 | c1c0f2a9 |
| ca7ae5c92135 | Baltimore Orioles @ Chicago White Sox | starter_missing_market_missing | mlb.pregame.analysis.starter_missing_market_missing.v1 | v1 | ab3bcdae |
| edabdf7ec8c8 | Chicago Cubs @ San Diego Padres | starter_missing_market_backed_depth | mlb.pregame.analysis.starter_missing_market_backed_depth.v1 | v1 | 6778eaa0 |
| 49ff76ef8dda | Colorado Rockies @ Miami Marlins | starter_missing_market_backed_depth | mlb.pregame.analysis.starter_missing_market_backed_depth.v1 | v1 | 6a2aff94 |

Evidence json: scratch (`<scratch>/sm_canary_evidence.json`), out-of-repo, NOT committed. `regimeAllowlisted=true`
here reflects explicit canary eligibility via the env override, NOT a `DEFAULT_ALLOWLIST` promotion.

## failures or blocked regimes

None among the target regimes -- both confirmed registry-authoritative. The only blocker was live-scheduled
input retrieval (full stack down), handled via the sanctioned fixture-based canary; no evidence was faked and the
representative-vs-live distinction is documented above.

## buyer-facing impact

None. Identical `SportsAnalysisResponse` shape on every run; no UX/copy/pricing/billing/auth/tenant/dashboard/
confidence change; the model received the live-identical prompt bytes (registry built them, proven byte-equal).

## recommendation

Both `starter_missing_market_missing` and `starter_missing_market_backed_depth` now have: real-soak-clean
equivalence, deterministic routing tests (Phase 3.2), AND runtime registry-authoritative confirmation under
explicit canary. They meet the bar for a **separate, explicit Default Allowlist Widening v1** slice. Recommended
caveat before promotion: re-confirm on at least one genuinely live-scheduled game (stack up) so the confirmation
is not solely fixture-based, OR accept fixture+real-soak evidence as sufficient and widen with a live-data soak
to follow.

## risks / deferred items

- Runtime confirmation was fixture-based (live stack down); a live-scheduled confirmation is deferred.
- `PromptRouteDecision` is logged + capturable in-process but not persisted to a read model; historical
  calibration grouping by recipe/version is still deferred (Prompt Provenance Read-Model Exposure).
- Allowlist widening remains a separate decision; real-soak-clean + runtime-confirmed is necessary, not
  automatic promotion.
- `starter_missing_market_backed` (single-book) remains representative-only -- not naturally available, not
  confirmed.

Related: [[phase-3-2-global-prompt-routing-hardening-v1]], [[starter-missing-regime-capture-v1]],
[[registry-authoritative-prompt-canary-v1]], [[multi-slate-regime-coverage-v1]].
