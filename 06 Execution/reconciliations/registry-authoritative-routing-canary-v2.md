---
title: "Registry-Authoritative Routing Canary v2"
type: "reconciliation"
date: "2026-07-03"
status: "complete (validation) -- paid-canary-ready for backed_depth"
project: "DAI"
slice: "Registry-Authoritative Routing Canary v2"
repos:
  dai: "test-only (committed local, unpushed)"
  dai-vault: "docs-only"
tags:
  - prompting
  - registry
  - routing
  - validation
related:
  - "02 Platform/decisions/0009-registry-routing-canary-ready.md"
  - "02 Platform/decisions/0007-prompt-route-attribution-contract.md"
  - "06 Execution/reconciliations/source-readiness-preflight-gate-v1.md"
---

# Registry-Authoritative Routing Canary v2

**slice:** validate the chain source ingredients -> observedDataRegime -> selectedDataRegime -> registry
recipe -> assembled prompt -> provenance, WITHOUT weakening the live pipeline. dry-run/shadow only; no model
calls; registry routing stays DEFAULT-OFF.
**status:** complete 2026-07-03T22:23Z. `dai` test-only (+2 dry-run tests); `dai-vault` docs-only.
**verification:** agent-service pytest 427/427 (registry canary 21/21). 0 model calls, 0 writes, routing off.

## current routing state

- **live path (default):** registry canary DEFAULT-OFF; every run takes the live prompt; `selectedDataRegime`
  null; `promptSource=live`; `observedDataRegime` stamped (Attribution Contract v1).
- **registry path (opt-in, env `DAI_MLB_REGISTRY_PROMPT_CANARY=1`):** the registry recipe may become the
  model input ONLY for an allowlisted regime AND ONLY after byte equality with the live prompt; any
  disabled/non-allowlisted/mismatch/assembly-failure falls closed to the live prompt with an explicit
  reason. **cardinal invariant: the model never receives bytes different from the live prompt.**
- allowlist (`DEFAULT_ALLOWLIST`, 4): starter_enriched_market_backed_depth, starter_enriched_market_missing,
  starter_missing_market_missing, starter_missing_market_backed_depth.

## registry inventory (manifest v2, sha256, all lifecycle=shadow_only)

10 recipes (base + starter overlay + market overlay), hash-verified on load. matrix by observedDataRegime:

| observedDataRegime | recipe | allowlisted | expected selectedDataRegime | expected promptRouteKey | selectedPromptPath | fallback if not selected | status |
|---|---|:-:|---|---|---|---|---|
| starter_enriched_market_backed_depth | ...starter_enriched_market_backed_depth.v1 | **YES** | (same) | recipe@v1::regime | registry | mismatch/assembly_error -> live | **READY (byte-identical proven)** |
| starter_enriched_market_missing | ...starter_enriched_market_missing.v1 | **YES** | (same) | recipe@v1::regime | registry | mismatch -> live | ready (selects or fails closed) |
| starter_missing_market_missing | ...starter_missing_market_missing.v1 | **YES** | (same) | recipe@v1::regime | registry | mismatch -> live | ready |
| starter_missing_market_backed_depth | ...starter_missing_market_backed_depth.v1 | **YES** | (same) | recipe@v1::regime | registry | mismatch -> live | ready |
| starter_enriched_market_backed | ...starter_enriched_market_backed.v1 | no | -- | unknown (live) | live | regime_not_allowlisted -> live | blocked (allowlist) |
| starter_missing_market_backed | ...starter_missing_market_backed.v1 | no | -- | unknown | live | regime_not_allowlisted | blocked |
| starter_named_market_backed / _missing / _backed_depth | ...starter_named_*.v1 | no | -- | unknown | live | regime_not_allowlisted | blocked |
| starter_asymmetric_market_backed_depth | ...starter_asymmetric_market_backed_depth.v1 | no | -- | unknown | live | regime_not_allowlisted | blocked (resolves the one-sided assembly_error class; not promoted) |
| other asymmetric/named combos | (no recipe) | no | -- | unknown | live | regime_not_allowlisted (before recipe lookup) | unsupported |

## dry-run results (no model calls)

`decide_model_prompt(enabled=True)` on real fixtures (`_enriched_starter()` + `_depth_market()`):

| regime | promptSource | selectedDataRegime | recipeId | version | assembledHash | selectedPromptPath | attributionStatus | msg==live |
|---|---|---|---|---|---|---|---|---|
| starter_enriched_market_backed_depth | **registry** | starter_enriched_market_backed_depth | ...backed_depth.v1 | v1 | sha256 (64 hex) | registry | complete | **YES** |
| starter_enriched_market_missing | registry OR live (fail-closed) | (same or null) | -- | -- | -- | registry/live | complete/-- | YES |

New tests assert the FULL `PromptRouteDecision` provenance for the registry-success path (prior tests only
checked the `to_info()` dict). observedDataRegime == selectedDataRegime on the registry path; livePromptTemplateKey
is null (registry, not the live marker); recipe id/version/hash are never fabricated (present because a recipe
truly assembled + proved byte-identical).

## live vs registry comparison (backed_depth)

- **prompt bytes: IDENTICAL.** the registry recipe (base + enriched overlay + backed_depth overlay) reproduces
  `build_mlb_user_message(...)` byte-for-byte (`test_select_allowlisted_uses_registry_byte_identical` +
  `test_registry_success_decision_full_provenance`: `msg == live`).
- **intended difference: none in bytes** -- only WHICH builder produced the (identical) bytes, plus provenance
  (`promptSource=registry`, `selectedDataRegime` populated, recipe id/version/hash).
- **prompt behavior change: NO** -- the model input is unchanged; the analyzer decision path is untouched.
- **risk: low.** the byte-equality guard means even a drifted recipe cannot reach the model (it fails closed).

## fail-closed behavior (all proven)

| condition | result | reason |
|---|---|---|
| registry disabled (default) | live, observedDataRegime stamped | fallbackReason=disabled |
| regime not allowlisted | live | regime_not_allowlisted |
| recipe assembly fails (partial evidence) | live, never raises | assembly_error (+ bounded detail, <=500 chars) |
| registry bytes != live bytes | live, logged loudly | mismatch (no misleading success provenance) |
| recipe valid + byte-identical | registry selected | full provenance populated |

## /rows + pooled readout (no change needed)

`/rows` already surfaces `promptSource`, `selectedDataRegime`, `selectedPromptRecipeId`, `selectedPromptVersion`,
`selectedPromptPath`, `attributionStatus`, `observedDataRegime` (Attribution Contract v1); pooled has `byRoute`
(selectedDataRegime) + `byObservedRoute`. Registry-routed runs are already distinguishable from live runs
(promptSource registry vs live; selectedDataRegime populated vs null). No additive field required; /metrics
denominator untouched.

## readiness decision: OPTION C -- PAID-CANARY-READY (backed_depth)

evidence: recipe exists + allowlisted; assembles; **byte-identical to live (proven)**; selection works
(promptSource=registry); full provenance populates; fail-closed on every error class with specific reasons;
/rows distinguishes registry runs. the byte-identity guard makes a paid canary carry **no prompt-behavior
risk** -- the model sees identical bytes; only provenance changes.

### approval packet (paid registry canary -- NOT run; needs explicit operator approval)

- **candidate:** one v8-eligible backed_depth game (screened `generationEligibleForTargetRegime=true` via
  `/source-readiness`), so the canary doubles as a real backed_depth ROUTE measurement row.
- **target observedDataRegime:** starter_enriched_market_backed_depth.
- **expected recipe:** `mlb.pregame.analysis.starter_enriched_market_backed_depth.v1` @ v1.
- **config:** `DAI_MLB_REGISTRY_PROMPT_CANARY=1` (default allowlist) set LOCAL/scoped for the run only; revert
  after. never committed to default dev/prod config.
- **max paid calls:** 1 (or up to the remaining v8 cap of 8 if routing the whole eligible cohort -- separate go).
- **expected cost:** ~$0.002 gpt-4o-mini + 1 odds PaidExternal call.
- **stop conditions:** if the run's `/rows` shows `promptSource=live` (fell back) instead of `registry`, or
  `selectedDataRegime != starter_enriched_market_backed_depth`, STOP and investigate (mismatch/assembly).
- **comparison vs live:** bytes proven identical in tests; on the run confirm `promptSource=registry` +
  `selectedDataRegime` populated + `selectedPromptRecipeId` correct + `attributionStatus=complete`.
- **settlement:** via the canonical residue contract when the game is Final.
- **residue verification:** `/rows` shows selectedDataRegime + promptSource=registry + recipeId + complete
  attribution; `/metrics` denominator moves only by the new reconciled outcome.

## v8 measurement recommendation

**split, evidence-first.** the registry canary and v8 CONVERGE: routing a v8-eligible backed_depth game via
registry gives it `selectedDataRegime=starter_enriched_market_backed_depth` (registry ROUTE attribution)
instead of `unknown`, with IDENTICAL bytes/decision. recommend: (1) when markets post (source-readiness
eligible), run ONE paid registry canary on a backed_depth game and confirm registry provenance on /rows;
(2) then decide whether to route the remaining eligible v8 games via registry (route-labeled) or keep them
live-path (unknown route, still moves the 0.80 confidence bucket). do NOT enable registry routing in default
config; the canary sets the env for its run only.

## safety ledger

paid model calls 0; new game runs 0; reconciliation writes 0; DB migrations 0; prompt text changed 0;
live-path prompt bytes changed 0; default prompt-selection behavior changed 0; registry routing default
changed 0 (still off); decision behavior 0; buyer-visible output 0; metrics denominator 0; historical rows
backfilled 0; v8 remaining calls spent 0 (still 8).
