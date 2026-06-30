# Registry Assembly Error Diagnostic v1 -- Evidence Report

**status:** complete (root-caused; no code change; attribution + diagnosability recommendations deferred)
**date:** 2026-06-30

## purpose

Root-cause the registry `assembly_error` that caused live batch run 260018 (Tigers@Yankees) to fail closed to
the live prompt, and decide whether it is a one-off partial-evidence case, an overlay coverage gap, or a prompt
registry assembly defect. Diagnostic + containment only -- no prompt rewrite, no cohort rerun, no paid calls.

## start state

- `dai`: clean, synced, `289777f` (0/0). `dai-vault`: clean, synced, `d6b475c` (0/0, pushed previous slice).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` in `registry_prompt_canary.py` unchanged -- exactly four regimes.
- No paid calls run this slice.

## run 260018 evidence

| field | value |
|---|---|
| AgentRunKey / RunId | 260018 / `36bd433e-f36b-1410-816e-00373db4b724` |
| game | Detroit Tigers @ New York Yankees, 2026-06-29, gamePk 823529 |
| Status | completed (lean=home) |
| StartedUtc | 2026-06-30 01:09:31Z |
| ErrorMessage | NULL (assembly_error is non-fatal; only logged) |
| InputJson | 103 chars -- `{Competition, HomeTeam, AwayTeam, GameDate}` only |
| OutputJson | 6722 chars (normal live-prompt artifact) |
| AgentRunOutcome | away_win 7-3 -> evaluation **incorrect** (reconciled 2026-06-30) |

Persisted `PromptRouteProvenanceJson`:

```json
{"promptSource":"live","registryAuthoritativeEnabled":true,"legacyFallbackUsed":true,
 "regimeAllowlisted":true,
 "routingReason":"registry assembly failed for regime starter_enriched_market_backed_depth; live prompt authoritative",
 "fallbackReason":"assembly_error",
 "selectedDataRegime":"starter_enriched_market_backed_depth",
 "selectedPromptRecipeId":null,"selectedPromptVersion":null,"assembledHash":null,
 "agentRunId":"36bd433e-...","competition":"mlb","createdUtc":"2026-06-30T01:09:32Z"}
```

Key: **`selectedDataRegime` IS retained** (`starter_enriched_market_backed_depth`). Only `selectedPromptRecipeId`,
`selectedPromptVersion`, and `assembledHash` are null -- because assembly raised before a recipe `result` existed.

## observed fallback behavior (verified against source)

Path: `registry_prompt_canary.py::decide_model_prompt`.
1. canary enabled, regime derived via `mlb_route_and_slots` -> `starter_enriched_market_backed_depth`.
2. regime is allowlisted (`regimeAllowlisted=true`), so assembly is attempted:
   `builder.assemble_recipe_for_migration(registry, route, slots, mode="shadow")`.
3. assembly raised `PromptAssemblyError` (`builder.py:108`). The canary caught it, logged
   `log.error(... "registry prompt canary assembly failed for regime %s")` (`builder.py:110`), and returned the
   **live prompt** via `_live("assembly_error", ..., enabled=True, regime=regime, allowlisted=True)`
   (`registry_prompt_canary.py:112-114`). `_live` carries the regime but no recipe_id/version (none exist yet).
4. The cardinal invariant held: the model received the **live prompt bytes**; the run completed normally and was
   later reconciled. Fail-closed worked exactly as designed.

`PromptAssemblyError` is raised inside `_compose_recipe` / `_render_template` (`builder.py:62-90, 132, 141`) for:
missing required slot, unknown provided slot, undeclared placeholder, a slot value with template braces or over
the length cap, an unresolved placeholder, or no selectable recipe.

## root cause

**Partial-evidence / overlay-coverage edge case at the regime-classification vs recipe-slot boundary.**

The regime classifier labels a game `starter_enriched` once **both starters are announced**, but the enriched
registry recipe requires **complete, symmetric enriched starter-quality slots**. When one starter's quality
evidence is missing/partial (asymmetric enriched evidence), the recipe's required slots are un-representable and
`_render_template` fails closed with `PromptAssemblyError` (missing/unknown slot). The **live** prompt is
free-form and tolerates the asymmetry, so it produces a normal artifact -- hence registry fails while live
succeeds for the same game.

This is exactly the shape the canary code comment names ("partial-evidence / un-representable input shape") and
the shape the existing regression test encodes:

> `tests/test_registry_prompt_canary.py::test_select_assembly_failure_falls_back` (line 146-148):
> "one-sided starter quality is an allowlisted regime by state but un-representable -> assembly fails" --
> `homeStarterQuality` present, away quality absent -> `PromptAssemblyError` -> live fallback.

Classification against the slice's cause taxonomy:

- missing/partial evidence field -- **YES** (one-sided starter quality).
- overlay coverage gap -- **YES** (enriched recipe does not cover asymmetric enriched evidence).
- source-depth classification edge case -- **YES** (regime says enriched; evidence is partial).
- registry recipe mismatch -- no (that is the separate `mismatch` fallback reason; not seen here).
- byte-equivalence strictness -- no (`mismatch`, not `assembly_error`).
- serialization/order/hash issue -- no.
- unrelated transient error -- no (deterministic for the input shape; would recur on identical evidence).

**Not a registry defect.** It is correct, deterministic containment for an evidence shape the strict recipe
cannot represent but the live prompt can.

## whether reproducible

- **Category: reproducible and already covered** by `test_select_assembly_failure_falls_back` (one-sided starter
  quality -> assembly fails -> live). No new test needed; adding one would duplicate existing coverage.
- **Exact 260018 instance: NOT reproducible** from persisted state. `InputJson` holds only teams+date; the
  runtime `starter_context`/`market_context` that produced the slots were not persisted; `ErrorMessage` is null;
  request-capture (`DAI_MLB_REQUEST_CAPTURE`) was off during the batch; no batch logs/scratch survive; the
  `PromptAssemblyError` message naming the failing slot was logged at runtime only and is gone.

## whether code changed

**No.** Fail-closed behavior is correct and tested; the buyer run completed and reconciled. No defect requires a
fix. Two improvements are recommended below but intentionally deferred to keep this slice diagnostic.

## attribution leak analysis (design question)

When registry assembly fails closed to live, should provenance retain only the actual route, the attempted
route, or a fallback attribution key?

Findings:
- The actual prompt used was **live** -- provenance must not pretend it was registry-authored. The frozen
  `PromptRouteDecision` contract (`extra="forbid"`, `_coherent` validator) already enforces this: a `live`
  source must carry `legacyFallbackUsed=true` + a `fallbackReason`. Honest today.
- The attribution loss is **not** in the data. `selectedDataRegime` + `fallbackReason` are already persisted.
  The loss is purely in the metrics grouping key: `PromptRouteCalibrationExport.RouteKey` (and its python
  mirror) returns `recipe@version::regime` **only when all three are non-empty**, else `"unknown"`. Because the
  fallback run has null recipe/version, its outcome buckets to `unknown` even though its intended regime is
  known -- so the enriched route shows n=7 instead of 8 and the fallback's (incorrect) outcome lands in
  `unknown`.

**Recommendation: Option C -- fallback attribution key, deferred to a dedicated minimal slice.**
- Keep `promptSource=live` (no pretense of registry authorship).
- When recipe/version are null but `selectedDataRegime` + `fallbackReason` are present, derive a
  fallback-specific key, e.g. `starter_enriched_market_backed_depth::assembly_error`, instead of `unknown`.
- No contract change, no DB change, no new provenance fields -- the data already exists; only `RouteKey`
  (C# + python mirror) + a couple of tests + the metrics aggregation buckets change.
- Rejected Option A (keep `unknown`): honest but loses recoverable calibration signal. Rejected Option B
  (add attempted recipe/version metadata): unnecessary -- recipe/version genuinely do not exist pre-assembly,
  and the regime alone is enough to attribute the failed route.

Deferred rather than done here because it changes the grouping semantics of a live metrics endpoint and should
land as its own scoped change with calibration-consumer review, not inside a diagnostic slice.

**Secondary recommendation (also deferred): persist the assembly_error detail.** Today the `PromptAssemblyError`
message (which slot failed) is logged but never persisted (`PromptRouteDecision` is `frozen`/`forbid`). The next
occurrence will be equally un-diagnosable. A future optional `fallbackDetail` field (or a captured failing-slot
list) would make root-cause possible without log archaeology. Defer -- it is a contract change.

## tests / verification run

All non-paid, no code changed:
- `pytest tests/test_registry_prompt_canary.py tests/test_migration_readiness.py -q` -> **28 passed**
  (includes `test_select_assembly_failure_falls_back` and the "assembly fails closed and NOTHING is captured"
  migration-readiness case).
- `dotnet test DevCore.Api.Tests --filter "PromptRouteProvenance|PromptRouteCalibration|AgentRunsController"`
  -> **90 passed** (includes `null_provenance_exports_unknown_route_safely`,
  `malformed_provenance_does_not_crash_and_is_unknown_route`).
- Source-verified: `registry_prompt_canary.py`, `builder.py`, `contracts.py` (PromptRouteDecision),
  `PromptRouteCalibrationExport.RouteKey`. DB-verified: 260018 row, provenance JSON, outcome.

## buyer-facing impact

**None.** Diagnostic only. No prompt template, recipe, allowlist, confidence, reconciliation rule, UX, copy, or
body change. No chain-of-thought or prompt text exposed.

## paid-call status

**None.** No model calls. DB/source/test inspection only.

## default allowlist status

**Unchanged** -- exactly four regimes.

## risks / deferred items

- The exact 260018 failing slot is unrecoverable (input not persisted). Future occurrences will be equally
  opaque until the assembly_error detail is persisted (secondary recommendation).
- Attribution leak (`assembly_error` -> `unknown` bucket) persists until Option C is implemented; the enriched
  route's match rate undercounts the one fallback game.
- Partial/asymmetric enriched evidence will keep failing closed to live for the enriched recipes. This is safe
  but means such games never exercise the registry path -- worth noting before scaling live cohorts, since a
  cohort heavy in partial-evidence games will under-represent registry-authoritative coverage.

## next recommended slice

**Calibration Route Attribution Fix v1** (Option C) -- minimal: change `RouteKey` (C# + python mirror) to emit
`{selectedDataRegime}::{fallbackReason}` for fallback runs with a known regime, add tests, no contract/DB change.
This directly closes the attribution leak this diagnostic identified.

If preferring breadth over the attribution fix: **Outcome Reconciliation Follow-up v1** (reconcile the 3
starter-missing soak games once Final -- non-paid, time-gated) is the next-best non-paid step.
