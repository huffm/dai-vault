---
title: "Phase 3.2 Global Prompt Routing Hardening v1"
type: "evidence-report"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Phase 3.2 Global Prompt Routing Hardening v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - provenance
  - prompt-registry
  - observability
related:
  - "06 Execution/reports/prompt-provenance-read-model-exposure-v1.md"
  - "06 Execution/reports/dotnet-agentrun-prompt-provenance-persistence-v1.md"
---

# Phase 3.2 Global Prompt Routing Hardening v1

**status:** active doctrine (shipped + verified; DEFAULT OFF; no real canary execution this slice)
**date:** 2026-06-29

## purpose

Make prompt selection on the registry-authoritative path explicit, explainable, and evidence-backed before any
wider registry-authoritative usage. Phase 3.1 built deterministic regime mapping and recipe selection; the
Registry-Authoritative Prompt Canary made the registry prompt a possible model input for two real-proven
regimes. What was missing was a single typed contract that records WHY a given prompt fed the model. Phase 3.2
adds that contract -- a `PromptRouteDecision` -- and proves it deterministically across every implemented
regime, without changing any buyer-facing behavior, model bytes, or the model-call count.

## problem it solves

The canary already chose live-vs-registry safely, but the decision existed only as an ad-hoc `info` dict
(`{"source": ..., "reason": ..., "regime": ...}`) that the analyzer discarded (`model_msg, _ = ...`). There was
no typed, closed, testable record of: which recipe was selected, its version, whether the canary was enabled,
whether the regime was allowlisted, whether the live/legacy prompt was used as fallback, and the explicit
fallback reason. Without that, a later calibration pass could not group outcomes by recipe/version, and routing
safety rested on prose rather than a contract.

## routing contract

The hardened decision is `app/prompting/contracts.py :: PromptRouteDecision` (frozen, `extra="forbid"` -- a
closed, tamper-evident value object). Fields, with the slice's intended names mapped to the implementation's
actual vocabulary:

| slice field                     | implementation field            | meaning                                                            |
| ------------------------------- | ------------------------------- | ------------------------------------------------------------------ |
| selectedPromptRecipeId          | `selectedPromptRecipeId`        | recipe id when registry-sourced (or named on a mismatch); else None |
| selectedPromptVersion           | `selectedPromptVersion`         | recipe/registry version when registry-sourced; else None           |
| selectedDataRegime              | `selectedDataRegime`            | derived regime; None when disabled (no routing work done when off)  |
| routingReason                   | `routingReason`                 | human-readable explanation of the decision                         |
| fallbackReason (nullable)       | `fallbackReason`                | closed set: disabled / regime_not_allowlisted / mismatch / assembly_error / error; None when registry-authoritative |
| registryAuthoritativeEnabled    | `registryAuthoritativeEnabled`  | whether the canary was enabled for this run                        |
| legacyFallbackUsed              | `legacyFallbackUsed`            | True whenever the live (legacy) prompt fed the model               |
| promptProvenance                | `assembledHash` (+ recipe/version/regime) | full PromptProvenance stays shadow-only; on the canary path the assembled hash is the integrity anchor |
| allowlist / canary eligibility  | `regimeAllowlisted`             | whether the derived regime was in the canary allowlist             |
| promptSource                    | `promptSource`                  | `live` or `registry` -- which builder produced the model bytes     |

`decide_model_prompt(...)` in `app/services/registry_prompt_canary.py` is the deterministic core: it returns
`(model_prompt, PromptRouteDecision)`. `select_model_prompt` is now a thin back-compat shim that projects the
legacy `info` dict via `PromptRouteDecision.to_info()`. `run_mlb_prompt_canary` returns the typed decision, and
the analyzer logs it (observability only).

The contract answers the six required questions:

1. **What evidence condition?** `selectedDataRegime` (derived by the same typed-evidence rules the live prompt
   branches on, via `mlb_route_and_slots`).
2. **Which recipe?** `selectedPromptRecipeId` + `selectedPromptVersion`.
3. **Why eligible?** `regimeAllowlisted` + `registryAuthoritativeEnabled` + `routingReason`.
4. **Was fallback used?** `legacyFallbackUsed` + `fallbackReason`.
5. **Registry-authoritative or legacy?** `promptSource` (`registry` only when proven byte-identical).
6. **Can a later pass group by recipe/version?** Yes -- `selectedPromptRecipeId` / `selectedPromptVersion` /
   `assembledHash` are the grouping keys.

## regime matrix

Nine implemented regimes (3 starter states x 3 market states). All nine are shadow/equivalence-proven
(byte-equal to the live prompt for representative inputs) and, as of this slice, each has a deterministic
routing test (`test_decision_regime_matrix_selects_own_recipe`, parametrized over all nine).

| regime                                | starter   | market       | equivalence-proven | routing-tested (3.2) | real-soak-clean | canary-confirmed | allowlisted |
| ------------------------------------- | --------- | ------------ | ------------------ | -------------------- | --------------- | ---------------- | ----------- |
| starter_enriched_market_backed_depth  | enriched  | backed_depth | yes                | yes                  | yes             | yes              | **yes**     |
| starter_enriched_market_missing       | enriched  | missing      | yes                | yes                  | yes             | yes              | **yes**     |
| starter_missing_market_missing        | missing   | missing      | yes                | yes                  | yes             | no               | no          |
| starter_missing_market_backed_depth   | missing   | backed_depth | yes                | yes                  | yes             | no               | no          |
| starter_enriched_market_backed        | enriched  | backed       | yes                | yes                  | no (single-book)| no               | no          |
| starter_missing_market_backed         | missing   | backed       | yes                | yes                  | no (single-book)| no               | no          |
| starter_named_market_backed           | named     | backed       | yes                | yes                  | no              | no               | no          |
| starter_named_market_missing          | named     | missing      | yes                | yes                  | no              | no               | no          |
| starter_named_market_backed_depth     | named     | backed_depth | yes                | yes                  | no              | no               | no          |

Status summary: **allowlisted = 2** (canary-confirmed, real-soak-clean enriched regimes); **real-soak-clean but
not allowlisted = 2** (the two starter_missing depth/missing regimes); **representative-only / under-tested = 5**
(3 named + 2 single-book backed -- not observed in real data, no real-soak evidence). Allowlist (2) is
deliberately narrower than real-soak-clean (4): widening is a separate explicit decision, not implied by
real-soak cleanliness.

## default-off / fallback posture

- The canary is DEFAULT OFF (`RegistryPromptCanaryConfig.enabled=False`). When off, `decide_model_prompt`
  returns the live prompt immediately and does NO routing work (`selectedDataRegime` stays None) -- the
  default-off path is byte-identical and untouched.
- Every non-registry-authoritative outcome returns the live prompt with an explicit `fallbackReason`: disabled,
  regime_not_allowlisted, mismatch, assembly_error, or error. The canary never raises into the live request.
- The registry prompt is used as the model input ONLY when enabled AND the regime is allowlisted AND the recipe
  assembles AND it equals the live prompt byte-for-byte. So the model can never receive bytes that differ from
  the live prompt; the decision only records WHICH builder produced those (proven-identical) bytes.

## model-call invariant

The analyzer still performs exactly one model call per run (`return await _call_model(...)`). The routing
decision is pure metadata derived from the single shadow compose already performed by the canary; it adds no
model call and does not change `model_msg`. Proven by `test_analyze_enabled_returns_decision_single_call` (one
call, live bytes) plus the pre-existing canary analyzer tests.

## what changed in code

- `app/prompting/contracts.py`: added `PromptRouteDecision` (frozen, closed) + `PromptSource` / `FallbackReason`
  literals + a `to_info()` legacy projection.
- `app/prompting/__init__.py`: exported `PromptRouteDecision`.
- `app/services/registry_prompt_canary.py`: added `decide_model_prompt` (typed-decision core) and a `_live`
  helper; reimplemented `select_model_prompt` as a back-compat shim over it; `run_mlb_prompt_canary` now returns
  the typed decision; an `assembly_error` fallback reason was separated from the generic `error` catch so a
  partial-evidence shape is reported distinctly while still retaining the regime.
- `app/services/sports_analyzer.py`: capture the decision from `run_mlb_prompt_canary` and log it (observability
  only; model bytes unchanged).
- `tests/test_prompt_route_decision.py`: new suite (17 tests) covering default-off, registry-authoritative
  selection, all three fallback reasons, the 9-regime routing matrix, contract immutability, legacy projection,
  and the single-call analyzer invariant.

## what did not change

No buyer-facing UX or copy; no pricing/billing/auth/tenant/dashboard; no new product surface; no model payload
shape; no prompt wording/templates/recipes/manifest (still 8 templates, 9 recipes); no allowlist change
(`DEFAULT_ALLOWLIST` still the same two regimes); no new prompt recipes; no confidence semantics; no .NET; no DB
schema; no new infrastructure or dependencies; no paid model calls.

## tests run

- `pytest tests/test_prompt_route_decision.py tests/test_registry_prompt_canary.py -q` -> 27 passed.
- `pytest tests/test_mlb_prompt_equivalence.py tests/test_migration_readiness.py tests/test_mlb_branch_overlays.py
  tests/test_shadow_validation.py tests/test_prompt_registry_contract.py tests/test_prompt_provenance.py -q` ->
  109 passed (legacy equivalence/canary protection intact).
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- `pytest -q` (full suite, final verification) -> 347 passed (was 330; +17 new), 0 failed.
- No paid model calls.

## risks / deferred items

- The decision is observability metadata; it is logged but not yet persisted to a read model, so a calibration
  pass cannot yet query routing decisions historically (deferred: prompt provenance read-model exposure).
- Five regimes remain representative-only (no real-soak evidence): 3 named + 2 single-book backed. They route
  deterministically but have not been observed in real data.
- The allowlist (2) is narrower than the real-soak-clean set (4) by design; widening to the two
  starter_missing regimes needs an explicit canary-confirmation + allowlist-widening slice.
- `decide_model_prompt` includes the recipe id/version on a mismatch decision for diagnosis; this is intentional
  (the bytes are still the live prompt) but a reader must not treat a mismatch decision as registry-authoritative
  (guard: `promptSource == "live"` and `legacyFallbackUsed is True`).

## next recommended slice

Starter-Missing Canary Confirmation + Allowlist Widening v1 (operator-approved, paid): runtime canary
confirmation for `starter_missing_market_missing` + `starter_missing_market_backed_depth` via the
`DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES` env override for the run only, then a separate explicit decision to
widen `DEFAULT_ALLOWLIST` to the four runtime-confirmed regimes. Alternatively, prompt provenance read-model
exposure so routing decisions are queryable for calibration.

Related: [[registry-authoritative-prompt-canary-v1]], [[live-migration-readiness-v1]],
[[multi-slate-regime-coverage-v1]], [[prompt-migration-closeout-2026-06-28]].
