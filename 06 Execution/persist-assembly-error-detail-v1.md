---
title: "Persist Assembly Error Detail v1"
type: "evidence-report"
date: "2026-07-01"
status: "complete"
project: "DAI"
slice: "Persist Assembly Error Detail v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - provenance
  - calibration
  - diagnostics
  - assembly-error
  - tdd
related:
  - "06 Execution/diagnostics/registry-assembly-error-diagnostic-v1.md"
  - "06 Execution/reports/calibration-route-attribution-fix-v1.md"
  - "02 Platform/decisions/0005-persist-assembly-error-detail.md"
---

# Persist Assembly Error Detail v1 -- Evidence Report

## purpose

Close the deferred enhancement from Registry Assembly Error Diagnostic v1: persist the concrete
`PromptAssemblyError` message so a future registry `assembly_error` fallback is diagnosable without log
archaeology. Adds an optional `fallbackDetail` string to prompt-route provenance, populated only on the
`assembly_error` path and surfaced read-only in the calibration `/rows` export. Diagnostic-only; no behavior
change otherwise. TDD-first.

## start state

- `dai`: clean, synced, `1a2ddf2` (0/0), then spec `c5269a3` + plan `f77e3b1` committed. `dai-vault`: clean,
  synced, `668e7ff` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4 regimes). No paid calls.
- Design + plan (brainstorming -> writing-plans) in `dai/docs/superpowers/specs/2026-07-01-persist-assembly-error-detail-design.md` and `dai/docs/superpowers/plans/2026-07-01-persist-assembly-error-detail.md`.

## field contract

- `fallbackDetail: string | null`; value = `str(PromptAssemblyError)` truncated to 500 chars
  (`_MAX_FALLBACK_DETAIL_LEN`).
- Populated ONLY when `fallbackReason == "assembly_error"`; null for every other row and every legacy row.
- Not a route-key or metrics-grouping input. Registry decisions are contractually forbidden from carrying it.

## code changes (6 touchpoints across both stacks, all additive)

**python (agent-service):**
1. `app/prompting/contracts.py` -- `PromptRouteDecision.fallbackDetail`; `_coherent` now rejects a `registry`
   decision carrying a `fallbackDetail`.
2. `app/services/registry_prompt_canary.py` -- `_MAX_FALLBACK_DETAIL_LEN=500` + `_truncate()`; `_live(detail=...)`;
   the `except PromptAssemblyError as exc` site passes `detail=_truncate(str(exc))`. The generic `error` catch is
   untouched (stays null).
3. `app/prompting/route_provenance.py` -- `PromptRouteProvenance.fallbackDetail` + copied in
   `build_route_provenance` (so the `X-Prompt-Route-Provenance` header carries it).
4. `app/services/route_calibration_export.py` -- `fallbackDetail` added to `CALIBRATION_FIELDS` + `_PROVENANCE_FIELDS`.

**C# (platform):**
5. `DevCore.AiClient/PromptRouteProvenance.cs` -- `FallbackDetail` (json `fallbackDetail`, trailing nullable). The
   controller's `SerializeRouteProvenance` re-serializes the whole record, so the durable
   `AgentRuns.PromptRouteProvenanceJson` column now persists it going forward; `TryParseHeader` reads it back.
6. `DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` -- `PromptRouteCalibrationRow.FallbackDetail`
   (trailing-optional, like `artifactVersion`), populated from `prov?.FallbackDetail` in `Shape`, surfaced by
   `GET /api/agent-runs/prompt-route-calibration/rows`.

## plan deviation (grounded, recorded)

The plan assumed the existing `test_select_assembly_failure_falls_back` fixture (one-sided starter quality)
reaches the assembly path. It does not: that input classifies as `starter_asymmetric_market_missing`, which is
not in the test allowlist, so it returns `regime_not_allowlisted` BEFORE assembly. (The old test only asserted
`source=="live"`, true for both paths, so the gap was invisible.) A real `assembly_error` for an allowlisted
regime is the rare production case -- partial evidence that still classifies as an allowlisted regime -- which is
not synthesizable without a live model run. The capture behavior was therefore tested deterministically by
monkeypatching `builder.assemble_recipe_for_migration` to raise `PromptAssemblyError`, exactly as the existing
`test_select_mismatch_falls_back_and_logs` forces a byte divergence. This pins the new behavior precisely.

## tests (TDD red -> green)

- python `tests/test_prompt_route_decision.py` (+4): decision accepts `fallbackDetail`; registry+detail rejected;
  default null; `build_route_provenance` projects it into the header json.
- python `tests/test_registry_prompt_canary.py` (+4): assembly_error decision captures the exact message; detail
  truncated to 500; non-assembly (`disabled`) fallback carries no detail; `_truncate` unit.
- python `tests/test_route_calibration_export.py` (+2): row includes `fallbackDetail` from provenance; null with
  no provenance.
- C# `PromptRouteProvenanceTests.cs` (+2): header with `fallbackDetail` parses; absent -> null (back-compat).
- C# `PromptRouteCalibrationExporterTests.cs` (+2): assembly_error provenance row exports the detail; no-provenance
  row exports null.

**Results:** python **60 passed** (3 target files); C# `dotnet build` **0 warn / 0 err**; C# targeted filter
(`PromptRouteProvenance|PromptRouteCalibration|PromptRouteKeyFallback|AgentRunsController`) **107 passed**.

## live no-write cross-check (dev, tenant 1, :5007)

- `GET /prompt-route-calibration/metrics` **byte-identical** to the pre-slice snapshot: totalRows 263,
  reconciledRows 84, noDecisionRows 10, matched 51, unmatched 33, matchRate 0.6071, registry/live/fallback 27/1/1.
  Confirms the new field does not touch metrics grouping.
- `GET /prompt-route-calibration/rows`: 263 rows, **all 263 now carry a `fallbackDetail` key, 0 non-null** --
  correct, because the one existing `assembly_error` run (260018) predates this slice and no new assembly_error
  run was created (no paid calls). The value accrues on the NEXT assembly_error, not retroactively.

## paid-call status

**None.** Unit/integration tests + a read-only metrics/rows endpoint check. No model call, no reconciliation
write, no DB write.

## buyer-facing impact

**None.** Provenance/export metadata only. `SportsAnalysisResponse`, artifact body, prompt text, and
chain-of-thought are untouched.

## untouched surfaces (verified)

Prompts / templates / recipes: untouched. `DEFAULT_ALLOWLIST`: unchanged (4). Buyer copy / routes: untouched.
DB schema: **no migration** (`PromptRouteProvenanceJson` is `nvarchar`; the header json gained one nullable
field). Route-key derivation + metrics aggregator: unchanged (proven by the byte-identical `/metrics`).
Reconciliation: untouched.

## files changed

- `dai` (code+docs): `contracts.py`, `registry_prompt_canary.py`, `route_provenance.py`,
  `route_calibration_export.py`, `PromptRouteProvenance.cs`, `PromptRouteCalibrationExport.cs` + their tests;
  `docs/superpowers/specs/...` + `docs/superpowers/plans/...`. 8 commits (`c5269a3` spec, `f77e3b1` plan,
  `d03af41`, `4c64fa0`, `aba1173`, `67974fd`, `1fa0a92`, `b2f9771`).
- `dai-vault` (docs-only): this doc + `02 Platform/decisions/0005-persist-assembly-error-detail.md` + handoff
  entry.

## risks / deferred items

- Run 260018 stays `fallbackDetail = null` (its input was never persisted; not back-fillable).
- `PromptRouteCalibrationRow` widened by one trailing nullable field; consumers must tolerate it (established
  additive pattern -- the prior slice added three).
- If structured slot data is ever needed, it can layer on top of the string without breaking it (deferred, YAGNI).

## next recommended slice

Reconciliation is still the live thread: **Outcome Reconciliation Follow-up v6** once StatsAPI shows 824818
(07-01) `Final` -- per-run `/{28bd433e}/outcome` (MultipleMatches expected). The first real `fallbackDetail`
value will appear whenever the next live registry `assembly_error` occurs.
