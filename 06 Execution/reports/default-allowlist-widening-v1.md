---
title: "Default Allowlist Widening v1"
type: "evidence-report"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Default Allowlist Widening v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - provenance
related:
  - "06 Execution/reports/starter-missing-registry-canary-confirmation-v1.md"
  - "06 Execution/reports/phase-3-2-global-prompt-routing-hardening-v1.md"
---

# Default Allowlist Widening v1

**status:** active doctrine (4 regimes now default registry-authoritative-eligible; narrow promotion)
**date:** 2026-06-29

## purpose

Cross the registry-authoritative promotion boundary for exactly two starter-missing regimes by adding them to the
default canary allowlist (`DEFAULT_ALLOWLIST`). After this slice the default allowlist treats four regimes as
canary-eligible without requiring a per-regime env override. This is a narrow promotion, not a prompt-design or
model-quality slice.

## promotion decision

Promote exactly two regimes:

- `starter_missing_market_missing`
- `starter_missing_market_backed_depth`

No other regime is promoted. The promotion is justified by the combination of: deterministic routing
(Phase 3.2), real-soak-clean equivalence ([[starter-missing-regime-capture-v1]]), and paid runtime
registry-authoritative confirmation ([[starter-missing-registry-canary-confirmation-v1]]).

## default allowlist before / after

Before (2 regimes):

```
("starter_enriched_market_backed_depth", "starter_enriched_market_missing")
```

After (4 regimes, exact):

```
("starter_enriched_market_backed_depth",
 "starter_enriched_market_missing",
 "starter_missing_market_missing",
 "starter_missing_market_backed_depth")
```

File: `services/agent-service/app/services/registry_prompt_canary.py :: DEFAULT_ALLOWLIST`. Ordering style
preserved (enriched pair first, then the two newly promoted starter-missing regimes). The default-off posture is
unchanged: `RegistryPromptCanaryConfig.enabled` still defaults to False; promotion only affects which regimes are
eligible WHEN the canary is enabled.

## evidence basis

- **Phase 3.2 deterministic routing** ([[phase-3-2-global-prompt-routing-hardening-v1]]): all 9 regimes route
  deterministically; both target regimes have routing tests and select their own recipe.
- **Real-soak-clean equivalence**: both regimes proved byte-identical to the live prompt on real captured inputs
  (2 real inputs each).
- **Paid runtime canary confirmation**: 4 paid `gpt-4o-mini` runs confirmed both regimes route
  `promptSource=registry`, `registryAuthoritativeEnabled=true`, `legacyFallbackUsed=false`, `fallbackReason=null`,
  one model call each, buyer artifact shape unchanged.

## explicit caveat

The runtime canary confirmation was **fixture-based, not live-scheduled** -- the full platform stack
(docker/`devcore-sql`/platform-api) was down at confirmation time, so `analyze_mlb` was driven directly with
representative starter-missing inputs (proven byte-equal to live) rather than a live-scheduled game. The paid
calls and routing decisions were real; the game identities were representative.

## why the promotion is still acceptable / remaining risk

Acceptable because the registry prompt is used as model input ONLY when it is proven byte-identical to the live
prompt for that request; promotion does not change model bytes, only which builder produces the (identical)
bytes. The two regimes have three independent evidence layers (deterministic routing, real-soak equivalence,
runtime registry-authoritative confirmation). Remaining risk: the runtime confirmation was fixture-based; a
genuinely live-scheduled confirmation is deferred (mitigated by the byte-equality guard, which fails closed to
the live prompt on any divergence, and by the canary remaining default-off).

## regimes intentionally NOT promoted

- `starter_enriched_market_backed` (single-book) -- representative-only, no real evidence.
- `starter_missing_market_backed` (single-book) -- representative-only, no real evidence.
- `starter_named_market_backed` -- representative-only (named starter state).
- `starter_named_market_missing` -- representative-only (named starter state).
- `starter_named_market_backed_depth` -- representative-only (named starter state).

These remain off and continue to fall back to the live prompt (`promptSource=live`,
`fallbackReason=regime_not_allowlisted`).

## tests run (venv python, from services/agent-service)

- `pytest tests/test_prompt_route_decision.py tests/test_registry_prompt_canary.py -q` -> 36 passed.
- `pytest tests/test_prompt_provenance.py tests/test_prompt_registry_contract.py -q` -> 46 passed.
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- `pytest -q` (full suite, final verification) -> 356 passed (was 348; +8 new), 0 failed.
- No paid model calls.

New/updated tests prove: the default allowlist is exactly the four promoted regimes (and not all 9); both
starter-missing regimes are now registry-authoritative under the DEFAULT config (no env override); the five
non-promoted regimes still fall back to live under the default; the explicit env override still narrows the
allowlist; the single-model-call invariant holds (existing analyzer canary tests).

## buyer-facing impact

None. No UX/copy/pricing/billing/auth/tenant/dashboard/confidence/model-selection change. The model still
receives live-identical prompt bytes; buyer artifact shape is unchanged.

## rollback plan

Smallest revert: remove the two added lines from `DEFAULT_ALLOWLIST` (restore the 2-regime tuple) and revert the
two test deltas. No data migration, no env change, no service restart logic involved. The env-override mechanism
is independent and untouched, so an operator can also narrow at runtime via
`DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES` without a code revert.

## risks / deferred items

- Runtime confirmation was fixture-based, not live-scheduled (see caveat); a live-scheduled soak is deferred.
- `PromptRouteDecision` is logged/capturable but not persisted to a read model; historical calibration grouping
  by recipe/version is deferred (Prompt Provenance Read-Model Exposure).
- The canary remains default-off; promotion only sets eligibility, so no behavior changes until an operator
  enables `DAI_MLB_REGISTRY_PROMPT_CANARY`.

## next recommended slice

Live-scheduled starter-missing soak (confirm both promoted regimes on genuinely live games when the stack is up),
OR Prompt Provenance Read-Model Exposure v1 (persist route decisions for calibration), OR a broader cohort rerun
grouped by prompt recipe/version.

Related: [[phase-3-2-global-prompt-routing-hardening-v1]], [[starter-missing-registry-canary-confirmation-v1]],
[[starter-missing-regime-capture-v1]], [[registry-authoritative-prompt-canary-v1]].
