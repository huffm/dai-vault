# decision 0007: prompt route attribution contract

**date:** 2026-07-03
**status:** accepted (implemented in `dai`; docs-only in vault)

## context

The v8 measurement cohort halted at its canary: SD@LAD (be49433e) had backed-depth DATA (multi-book
market, enriched starter, confidence 0.80) but `selectedDataRegime = null`, `promptRouteKey = unknown`,
`promptSource = live`. Root cause (not a defect): the Registry-Authoritative Prompt Canary is DEFAULT-OFF
(`registry_prompt_canary.py`); when disabled, `decide_model_prompt` short-circuits BEFORE computing any
regime, so no `selectedDataRegime` is stamped and the pooled tool (which groups by `selectedDataRegime`)
buckets the run as `unknown`. The historical 16 `enriched_market_backed_depth` rows were produced in a
prior window when routing was enabled.

Consequence: a live-path artifact was not measurement-grade -- it could not say which observed data regime
and prompt path produced it. Turning registry routing on to fix this would be a prompt-SELECTION change
(forbidden), so attribution had to be added as pure OBSERVABILITY.

## decision

Stamp complete route attribution on every artifact, additively and non-semantically, WITHOUT enabling
registry routing or changing prompt selection.

1. **observedDataRegime** -- the regime the ACTUAL inputs support, classified deterministically from the
   same typed starter/market evidence the router uses (`migration_readiness.observed_data_regime`, reusing
   `_starter_state`/`_market_state`/`mlb_regime`). Computed for EVERY run (incl. the default-off live path)
   in `decide_model_prompt`; guarded so it can never break the request. It equals `selectedDataRegime` when
   routing is enabled.
2. **selectedDataRegime stays SELECTION-only** -- null on the live path. It is NOT overloaded to carry the
   observed regime, so it never implies a registry recipe was selected when it was not.
3. **selectedPromptPath** -- "registry" | "live" | "fallback": which construction path fed the model.
4. **livePromptTemplateKey** -- a truthful marker for the live/legacy hardcoded template
   (`mlb.matchup.analysis.live`); NEVER a fabricated registry recipe id; null on the registry path.
5. **attributionStatus / attributionReason** -- "complete" (registry recipe selected) | "partial"
   (observed-only, live path) | "unattributed" (observed regime could not be classified) + a reason string.

These ride the existing decision -> `PromptRouteProvenance` -> `X-Prompt-Route-Provenance` header ->
.NET run-row read model -> `/rows` path. `/rows` surfaces all five additively (trailing-optional,
metrics-ignored). The pooled tool gains an ADDITIVE `byObservedRoute` grouping (keyed on
`observedDataRegime`) alongside the unchanged `byRoute` (keyed on `selectedDataRegime`); no count or
denominator changes.

## non-semantic guarantee (proven by tests)

- **prompt text:** unchanged -- no template touched; `decide_model_prompt` still returns `live_prompt`
  byte-for-byte (asserted `prompt == "LIVE"`).
- **model input / call count:** unchanged -- observed regime is classified from already-retrieved contexts;
  no new model call.
- **prompt selection:** unchanged -- registry routing stays DEFAULT-OFF; the observed classifier selects
  nothing (recipe/version/hash stay null on live).
- **analyzer decision / confidence / advertised strength / buyer output:** unchanged -- attribution is
  run-row-adjacent metadata (header + `/rows` dev export), never the buyer artifact body.
- **denominators:** `/metrics` unchanged; `/rows` gains trailing nullable fields; `byObservedRoute` is
  additive.

## consequences

- Live-path runs are now measurement-grade: `observedDataRegime` attributes them to their real data regime
  (e.g. `starter_enriched_market_backed_depth`) via `byObservedRoute`, while `selectedDataRegime` honestly
  stays null (no registry selection happened). Calibration can read observed-route accuracy; prompt-selection
  review can read `selectedPromptPath` / `promptSource` / recipe fields separately.
- **v8 can resume** without enabling registry routing: the remaining backed-depth games, generated under the
  unchanged pipeline, will carry `observedDataRegime = starter_enriched_market_backed_depth` and land in the
  0.80 bucket -- attributable via `byObservedRoute`. (Historical rows, incl. canary be49433e, are NOT
  backfilled; the field is null on runs generated before this slice.)
- Enabling registry-authoritative routing (to make `selectedDataRegime` non-null on these runs) remains a
  separate, explicitly-approved prompt-selection slice; it is NOT required for measurement.

## references

- Origin: `06 Execution/reconciliations/v8-cohort-execution-canary-halt-v1.md` (the halt) and this slice
  `06 Execution/reconciliations/prompt-route-attribution-contract-v1.md`.
- Code: `services/agent-service/app/services/migration_readiness.py` (`observed_data_regime`),
  `registry_prompt_canary.py` (stamping), `app/prompting/contracts.py` + `route_provenance.py` (fields),
  `app/services/pooled_calibration.py` (`byObservedRoute`); `platform/dotnet/DevCore.AiClient/
  PromptRouteProvenance.cs` + `DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs` (/rows surface).
- Pattern lineage: additive `/rows` fields (ADR 0005, ADR 0006).
