# Prompt Registry Contract v1

**date:** 2026-06-28
**status:** IMPLEMENTED (NON-LIVE) + VERIFIED. First code contracts for controlled dynamic prompt assembly: typed
route context, template metadata, manifest + SHA-256 verification, deterministic fail-closed route selection, and
assembly-result metadata. NOT wired into the live analyzer path; `sports_analyzer.py` is unchanged. No prompt/model/
confidence tuning; no cohort work.
**type:** Phase 2 foundation slice. dai-slice-runner + dai-skill-router gate + test-driven-development +
dai-test-discipline + systematic-debugging + verification-before-completion + dai-grill-with-vault.
**anchor:** make prompt selection eventually become: typed workflow context -> approved template id/version ->
hash-verified template -> deterministic assembly metadata -- without routing any production analyzer call through it yet.

See also: [[controlled-dynamic-prompt-assembly-architecture-v1]] (the design this implements),
[[canonical-decision-composition-hardening-v1]], [[analyzer-generation-side-lean-agreement-hardening-v1]].

## 1. Purpose

Provide the contract + validation foundation for a future Prompt Assembly Engine: an allowlisted, versioned,
hash-verified registry of prompt templates selectable by typed facts only, fail-closed on anything unknown.

## 2. Relationship to Controlled Dynamic Prompt Assembly Architecture v1

This is **Phase 2** of that architecture. Phase 1 defined the design (Registry / Router / Builder / Contract /
Validator / future Protocol Runner). This slice implements the **contracts + registry/router selection + hash
verification** -- the Builder (real assembly) and live migration are Phase 3 (byte-equivalent MLB assembly).

## 3. What was implemented (all in `dai`, non-live)

New package `services/agent-service/app/prompting/`:
- `contracts.py` -- pydantic v2 models: `PromptRouteContext`, `PromptTemplate`, `PromptManifest`,
  `PromptSelectionResult`, `PromptAssemblyResult` (+ `TemplateLifecycle` / `RouteMode` literals).
- `hash_verifier.py` -- `sha256_file`, `verify_template_hash`, `TemplateHashError`.
- `manifest.py` -- `load_manifest` (+ `ManifestValidationError`): parse, contract-validate, reject wrong hash
  algorithm / duplicate id+version, and hash-verify every template file.
- `registry.py` -- `PromptRegistry.load(...)`, `.select(ctx, mode, allow_deprecated)`, `.to_assembly_result(...)`.
- `templates/manifest.json` + one non-live `shadow_only` template
  `mlb.pregame.analysis.current_equivalent.v1.txt` (a future migration target; NOT the live prompt).
- `tests/test_prompt_registry_contract.py` (17 tests).

## 4. What remains non-live

`sports_analyzer.py` and the FastAPI analyze path are unchanged -- the live MLB prompt still builds via the existing
f-strings. Nothing imports the registry on the request path. The example template is `shadow_only` and cannot be
selected in `live` mode. No `.NET` change.

## 5. Contract shapes

- **PromptRouteContext** (frozen, `extra='forbid'`): product, workflow, sport, competition, station, decisionType,
  dataRegime, outputSchemaId (required); microAction, sourceDepth{signal->level}, marketAvailable, identityConfidence,
  buyerReadiness, allowedTools (optional). `extra='forbid'` is the security boundary -- arbitrary prompt text cannot be
  smuggled in as a selector. tenantId intentionally omitted (no tenant/auth work in this layer).
- **PromptTemplate** (frozen): templateId, version, lifecycle (active|shadow_only|deprecated|disabled), templateType,
  filePath, sha256, requiredRouteFacts{field->value}, requiredSlots, outputSchemaId, allowedTools, approvedReviewRef,
  description.
- **PromptManifest**: manifestVersion, hashAlgorithm, templates[].
- **PromptSelectionResult**: selected (PromptTemplate|None), routeContext, selectionReason, failureReason, failClosed.
- **PromptAssemblyResult**: templateId, version, templateHash, routeContext, outputSchemaId, requiredSlots,
  assemblyReason, lifecycle; assembledHash + providedSlots are None in v1 (no real assembly yet).

## 6. Manifest / hash rules

SHA-256 over exact template file bytes. `load_manifest` fails closed on: invalid JSON, contract violation
(`extra='forbid'`), unsupported `hashAlgorithm` (only `sha256`), duplicate `(templateId, version)`, missing template
file, or hash mismatch. The recorded `sha256` in the manifest must equal the file's hash or the registry refuses to
load. (Recommended follow-up: a CI check asserting manifest hashes == file hashes, per the architecture doc.)

## 7. Routing behavior

Deterministic, no fuzzy matching, no ranking, no model/user input. `select(ctx, mode, allow_deprecated)`:
- a template matches only if **every** `requiredRouteFacts` entry equals the matching typed field on `ctx` (an
  unknown required key never matches -- fail closed).
- lifecycle gating: `disabled` never selectable; `deprecated` only with explicit `allow_deprecated` (test-only);
  `shadow_only` only in `mode="shadow"`; `active` always.
- exactly one selectable match -> selected; zero -> fail closed (no-match or lifecycle-excluded reason); more than one
  -> fail closed (ambiguous, never silently pick). Selection is identical across calls for the same context.

## 8. Security rules (enforced or contract-level)

Allowlisted templates (manifest); versioned (id@version, new version = new file); file hash verified; selection by
typed route facts only; frontend/user text cannot select (no free-text selector field; `extra='forbid'`); model
output cannot select (selection consumes facts, not responses); disabled fail closed; shadow_only excluded from live;
required slots declared on the template; outputSchemaId carried through; no secrets in the template file; future
assembly must fill typed slots (no raw provider payload dumps). The example template is explicitly non-live/shadow.

## 9. Tests

`services/agent-service/tests/test_prompt_registry_contract.py` (17): manifest loads; hash verify ok; hash mismatch
fails; duplicate id/version fails; unknown route fails closed; matching route selects expected; disabled not
selectable; deprecated not selectable in live (opt-in only); shadow_only only in shadow; missing file fails;
deterministic selection; required facts enforced; route context rejects arbitrary prompt text; outputSchemaId carried
into assembly result; assembly metadata complete (id/version/64-hex hash/reason; assembledHash None); invalid JSON
fails; wrong hash algorithm fails.

Commands + results:
- `cd services/agent-service && .venv/Scripts/python.exe -m pytest tests/test_prompt_registry_contract.py -q` -> **17 passed**.
- `.venv/Scripts/python.exe -m pytest -q` (full agent-service suite) -> **143 passed, 0 failed** (126 prior + 17 new).
- `sports_analyzer.py` confirmed UNCHANGED (git status); `.NET` untouched (no dotnet test required).

## 10. Migration path toward Prompt Assembly Engine v1 (Phase 3)

1. Add an `active` MLB template whose body, when its slots are filled, is **byte-equivalent** to today's
   `analyze_mlb` prompt (golden-equality test as the gate).
2. Implement the Builder: fill typed slots from the existing typed context objects (no new data).
3. Behind a shadow flag, assemble via the registry and diff against the live f-string output; ship only on exact
   equality. Then Phase 4 adds per-call audit logging (templateId/version/hash/routeContext/outputSchema).

## 11. Risks / open questions

- Registry currently lives in Python (where the model call is); the architecture doc leaves open whether the
  **router/contract** belongs in .NET (where route facts are computed at retrieve time) with a shared manifest. Resolve
  before Phase 4.
- No CI hash-check yet -- a manifest hash could drift from the file without failing build until load. Add in Phase 2.x.
- `dataRegime` is a free string today; a canonical `dataRegime`-from-`sourceDepth` mapping is still needed (Phase 2.x/3).
- Ambiguous-match handling fails closed by design; real multi-template recipes (layered overlays) will need an
  explicit composition order, not single-template match -- a Builder concern for Phase 3.

## 12. Recommended next slice

**Phase 3 -- Prompt Assembly Engine v1:** implement the Builder + an `active` byte-equivalent MLB template and a
golden-equality test proving the assembled prompt equals the current live prompt, still behind a shadow flag (no live
behavior change). Defer audit logging (Phase 4) and any protocol-as-execution.
