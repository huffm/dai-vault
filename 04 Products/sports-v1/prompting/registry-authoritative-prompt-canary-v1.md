# Registry-Authoritative Prompt Canary v1

**status:** active doctrine (shipped + verified; DEFAULT OFF; no real canary execution this slice)
**date:** 2026-06-28

## purpose

Allow the registry-built prompt to become the MODEL INPUT for MLB analysis, but only as a tightly-bounded
canary: default off, restricted to regimes with clean real-soak evidence, and only after the registry prompt is
proven byte-identical to the live prompt for that request. This is the first place the registry prompt can reach
the model, so the design centers on one invariant: the model never receives bytes different from the live
prompt.

## problem it solves

Phases 4 -> Migration Readiness -> Shadow-Parallel Routing -> Cohort Soak -> Live-Data soak proved the registry
prompt reproduces the live prompt byte-for-byte (representative, then real). The remaining step toward migration
is to let the registry prompt actually drive the model -- without risk. This canary does that for two
real-proven regimes only, with a live fallback and an equality guard, so it can be exercised safely before any
broad migration.

## strategic fit

The first controlled step of prompt-registry migration. It converts "the registry matches the live prompt" into
"the registry can BE the prompt" for proven-clean regimes, while keeping the live builder as the authority and
the safety floor.

## mental model

`select_model_prompt` chooses the model input. The live prompt is the default and the floor. The registry
prompt is used ONLY when: enabled AND regime is allowlisted AND it assembles AND it equals the live prompt
byte-for-byte. Because the registry prompt is used only when byte-equal, the model input is ALWAYS the same
bytes as the live prompt; the canary only changes which builder produced them.

## what it is

- `app/services/registry_prompt_canary.py`:
  - `DEFAULT_ALLOWLIST = ("starter_enriched_market_backed_depth", "starter_enriched_market_missing")` -- exactly
    the two regimes with clean real-soak evidence; never all 9.
  - `RegistryPromptCanaryConfig(enabled=False, regimes=DEFAULT_ALLOWLIST)` + `load_*` reading
    `DAI_MLB_REGISTRY_PROMPT_CANARY` / `DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES`.
  - `select_model_prompt(...) -> (model_prompt, info)` -- the core selector (live by default; registry only
    after the equality guard).
  - `run_mlb_prompt_canary(...)` -- analyzer-facing wrapper; constructs registry+builder only when enabled.
- `analyze_mlb` gains a keyword-only `canary_config`; a guarded block selects `model_msg` (default `user_msg`)
  before the single `_call_model(model_msg, ...)`.

## what it is not

- NOT broad registry-authoritative routing (2 regimes only). NOT a second model call. NOT a second artifact.
  NOT buyer-facing. NOT prompt tuning. No template/manifest/.NET change. No DB persistence.

## why only two regimes are allowlisted

They are the only regimes with clean REAL-soak evidence. [[real-cohort-live-soak-v1]] ran the real 2026-06-28
MLB slate (15 games): 15/15 matched, 0 failures, GO, across exactly these two regimes
(`starter_enriched_market_backed_depth` x2, `starter_enriched_market_missing` x13). The other 7 regimes are
proven only on representative fixtures, so they are excluded until they have real-soak evidence.

## approved uses

- Enable on a controlled/canary slice for allowlisted regimes only; the registry prompt then drives the model
  for those requests, byte-identical to the live prompt.

## disallowed uses

- Do not enable by default. Do not widen the allowlist beyond regimes with real-soak evidence. Do not treat a
  canary success as license for broad migration.

## cardinal invariant + equality guard

The model input is always byte-identical to the live prompt. `select_model_prompt` returns the registry text
ONLY at the single return that is downstream of `if registry_text != live_prompt:` (a strict, no-normalization
comparison against the passed-in live `user_msg`). Every other path -- disabled, non-allowlisted regime,
assembly failure (partial evidence), mismatch, any exception -- returns the live prompt. Verified
ADVERSARIALLY across 10 attack paths (disabled, allowlisted+equal, allowlisted+mismatch, non-allowlisted,
assembly failure, any exception, allowlisted-but-divergent, unchecked-return, allowlist default, concurrency):
the invariant holds on all 10. Both the inner selector and the outer analyzer block fail closed to live.

## fallback behavior

- Disabled (default): `model_msg = user_msg`; the canary is never consulted.
- Enabled, regime NOT allowlisted: live prompt (info.reason = `regime_not_allowlisted`).
- Enabled, allowlisted, registry assembles but != live: live prompt + loud `MISMATCH` error log; no misleading
  success provenance (none is emitted here at all).
- Enabled, allowlisted, assembly raises / any error: live prompt (info.reason = `error`).

## exact config / env flags

- `DAI_MLB_REGISTRY_PROMPT_CANARY` = `1|true|yes|on` (default off).
- `DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES` = comma-separated allowlist override. ABSENT/blank -> the hardcoded
  `DEFAULT_ALLOWLIST` (the safe, simple choice -- the 2 proven regimes, never all 9). A present value is the
  operator's explicit override; even then, every regime still passes the equality guard, so an over-broad
  allowlist cannot leak non-live bytes.

## truth hierarchy

The live `build_mlb_user_message` remains authoritative and is the safety floor. The registry prompt is used
only when it is provably identical to that authority. The canary is never above the live prompt.

## source or vault references to verify

- Selector + invariant: `services/agent-service/app/services/registry_prompt_canary.py` (`select_model_prompt`
  equality guard; `DEFAULT_ALLOWLIST`).
- Analyzer wiring: `services/agent-service/app/services/sports_analyzer.py` `analyze_mlb` (single
  `_call_model(model_msg, ...)`).
- Registry build reused: `app/prompting/builder.py` `assemble_recipe_for_migration`; regime via
  `app/services/migration_readiness.py` `mlb_route_and_slots`.
- Tests: `services/agent-service/tests/test_registry_prompt_canary.py` (10 cases).
- Evidence for the allowlist: [[real-cohort-live-soak-v1]].

## example usage

```
# enable the canary on the agent-service (allowlisted regimes only, default allowlist):
# DAI_MLB_REGISTRY_PROMPT_CANARY=1
# drive a real allowlisted mlb request via the platform; the model receives the registry prompt ONLY if it is
# byte-identical to the live prompt, else the live prompt. one model call, byte-identical input either way.
```

## whether any real canary execution was performed

NO. The optional real run needs explicit operator approval for a paid model call; it was not granted this slice.
The byte-identity invariant is fully proven offline (tests + adversarial review), so a real run is low-risk but
was not executed.

## acceptance criteria

- Disabled = byte-identical model input (verified).
- Enabled allowlisted + equal = registry prompt as model input, byte-identical to live (verified).
- Non-allowlisted / mismatch / error = live prompt (verified).
- Single model call; no second artifact; no buyer change (verified).
- Allowlist = exactly 2 proven regimes by default (verified).

## risks and failure modes

- **Allowlist scope.** Only 2 regimes are real-proven; widening requires real-soak evidence per regime
  ([[real-cohort-live-soak-v1]] gap: 7 regimes fixture-only).
- **Per-call registry build when enabled.** `run_mlb_prompt_canary` loads registry + builder per request when
  enabled (acceptable for a canary; cache deferred).
- **No real-run confirmation yet.** End-to-end behavior under a real enabled call is unverified by execution
  (only by tests + review).

## deferred decisions

- Real canary execution (needs paid approval).
- Expanding the allowlist to the other 7 regimes (needs real-soak evidence).
- Broad registry-authoritative routing (off the table until the allowlist is complete and a canary soak is
  clean). DB persistence; `.NET sourceDepth -> dataRegime` ownership; partial-evidence overlays -- unchanged
  deferrals.

## related docs

- [[real-cohort-live-soak-v1]] (allowlist evidence), [[mlb-analyzer-request-capture-v1]],
  [[live-prompt-routing-shadow-parallel-v1]] (shadow validation -- the compare-only neighbour),
  [[live-data-shadow-soak-v1]].

## recommended next slice

**Registry Canary Real Confirmation v1 (operator-approved, 1 paid call):** with the canary enabled, run one real
allowlisted MLB request and confirm HTTP 200, model input registry-selected but byte-identical, no mismatch, no
second call. Then **Multi-Slate Regime Coverage v1** to earn real-soak evidence for the other 7 regimes before
widening the allowlist. Only a complete, clean allowlist should move toward broad Registry-Authoritative Prompt
migration.
