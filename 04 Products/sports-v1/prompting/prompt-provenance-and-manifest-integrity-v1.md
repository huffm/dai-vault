# Prompt Provenance + Manifest Integrity v1

**date:** 2026-06-28
**status:** IMPLEMENTED (NON-LIVE) + VERIFIED. Adds (a) a shadow-only `PromptProvenance` audit record emitted next to a
recipe assembly, and (b) a dedicated, fail-collecting `verify_manifest_integrity` verifier + CI script. NOT wired into
the live analyzer; `sports_analyzer.py` unchanged. No prompt/model/confidence tuning; no cohort work; no DB persistence.
**type:** Phase 4 foundation slice. test-driven-development + verification-before-completion + code-review (high). The
dai-specific skills referenced in `dai/CLAUDE.md` (dai-skill-router/dai-slice-runner/dai-test-discipline/
dai-grill-with-vault/dai-agent-handoff) are NOT installed in this environment; superpowers equivalents + a manual
structured handoff were substituted (recorded in the Skills Gate note of the handoff).
**anchor:** make a shadow recipe assembly auditable (provenance) and make the manifest allowlist CI-verifiable (integrity)
before any audit-log persistence or live migration -- still shadow-only, prompt bytes byte-identical.

See also: [[market-depth-starter-named-overlays-v1]] (Phase 3.2), [[data-regime-overlay-mlb-prompt-equivalence-v1]]
(Phase 3.1), [[prompt-assembly-engine-v1]] (Phase 3), [[prompt-registry-contract-v1]] (Phase 2),
[[controlled-dynamic-prompt-assembly-architecture-v1]] (Phase 1).

## 1. Purpose

Two non-live additions the recommended Phase 4 next-slice called for:
1. **Prompt provenance** -- a deterministic, immutable audit record describing one shadow recipe assembly (recipe +
   regime + state dimensions + piece ids/hashes + assembled hash + route context + output schema + lifecycle + mode +
   timestamp), so a shadow assembly is fully auditable.
2. **Manifest integrity** -- a dedicated verifier that checks the whole allowlist and reports EVERY defect at once
   (missing file, stale hash, duplicate id, invalid piece reference, invalid json, wrong algorithm, missing manifest),
   runnable as a CI command.

## 2. What was implemented (all in `dai`, non-live)

- `app/prompting/provenance.py` (NEW) -- `PromptProvenance` (frozen, `extra='forbid'` pydantic model) + `build_provenance`
  (pure function over a selected recipe + the assembly result) + `utc_now_iso` (deterministic-shape UTC iso).
- `app/prompting/builder.py` -- `PromptBuilder.assemble_recipe_with_provenance(...)` returns `(result, provenance)`;
  it fails closed unless `mode == "shadow"`. Refactored a private `_build_recipe_result` so the recipe and the result's
  pieceHashes come from the SAME single selection (no double-select; provenance can never misalign with the result).
- `app/prompting/dataregime.py` -- `split_mlb_regime` (inverse of `mlb_regime`; fail-closed parse of the canonical
  `starter_{s}_market_{m}` regime).
- `app/prompting/integrity.py` (NEW) -- `ManifestIntegrityIssue`, `ManifestIntegrityReport`, `verify_manifest_integrity`
  (fail-collecting), and a `main(argv)` CLI (exit 0 clean / 1 defective).
- `scripts/check_prompt_manifest.py` (NEW) -- CI-friendly wrapper over `integrity.main` with clean output (the
  `python -m app.prompting.integrity` form prints a benign runpy double-import warning because the module is exported
  from the package; the script avoids it).
- `app/prompting/__init__.py` -- exports the new symbols.
- `tests/test_prompt_provenance.py` (29) and `tests/test_manifest_integrity.py` (12).

## 3. What remains non-live

`sports_analyzer.py` and the FastAPI analyze path are unchanged (git-confirmed). Nothing on the request path imports
provenance or integrity. All recipes are `shadow_only`; `assemble_recipe_with_provenance(..., mode="live")` fails closed.
No DB persistence (see sec 7). No `.NET` change.

## 4. Provenance model (`PromptProvenance`)

Fields captured: `recipeId`, `recipeVersion`, `dataRegime`, `starterState`, `marketState`, `pieceIds`, `pieceHashes`,
`assembledHash`, `routeContext`, `outputSchemaId`, `lifecycle`, `mode`, `createdUtc`.

- `pieceIds` from the recipe; `pieceHashes` (template-level, slot-independent) and `assembledHash` (sha-256 of the final
  rendered prompt text, slot-sensitive) from the assembly result the builder produced.
- `starterState`/`marketState` are derived from the canonical `dataRegime` (authoritative per `dataregime.py`); any
  explicit recipe metadata is cross-checked and a disagreement fails closed (a manifest defect).
- `assembledHash` is passed through; a missing one fails loud at the model rather than recording an entry with no
  integrity anchor.
- Frozen + `extra='forbid'`: the record is tamper-evident and no extra field can be smuggled in.

## 5. Timestamp strategy (deterministic + testable)

`createdUtc` is injectable: `assemble_recipe_with_provenance(..., created_utc=...)` and `build_provenance(...,
created_utc=...)` accept a fixed value for fully reproducible records (tests/goldens pass `"2026-06-28T00:00:00Z"`); when
omitted it defaults to `utc_now_iso()` (fixed-shape `...Z`, second precision). This keeps provenance deterministic given
its inputs and avoids a hidden wall-clock dependency in tests.

## 6. Provenance does not alter rendered prompt bytes (proven)

`build_provenance` reads only the recipe + the already-computed result; it never re-renders. Tests assert, for ALL 9 MLB
regimes, that `render_recipe(...) == build_mlb_user_message(...)` (the live oracle) byte-for-byte AND that
`assemble_recipe_with_provenance(...)` reports the same `assembledHash` as plain `assemble_recipe(...)`. Provenance is
emitted only for shadow assembly: `mode="live"` fails closed (`PromptAssemblyError`), so live routing can never emit it.

## 7. No DB persistence in v1 (deliberate)

Per the slice constraint, provenance is returned in-memory next to the assembly result. No existing shadow-metadata sink
is clearly correct yet (the .NET `AgentRunExecutionResult` persists the *artifact*, not the shadow prompt assembly), so
persisting now would force a premature schema decision. Persisting provenance to an audit sink is the next slice once a
correct sink is chosen.

## 8. Manifest integrity verifier

`verify_manifest_integrity(manifest_path) -> ManifestIntegrityReport`. Unlike `load_manifest` (fail-fast, typed
exceptions, registry path), the verifier is fail-collecting: it returns `ok` plus a list of every issue. Defect kinds:
`missing_manifest`, `invalid_json`, `invalid_structure`, `wrong_algorithm`, `duplicate_template`, `duplicate_recipe`,
`missing_file`, `stale_hash`, `invalid_piece_reference`. Tests prove each failure mode, that multiple issues are
collected at once (not fail-fast), that the real shipped manifest is clean (8 templates / 9 recipes), and the CLI exit
codes (0 clean / non-zero defective).

CI command (from `services/agent-service`): `python scripts/check_prompt_manifest.py` (or
`python -m app.prompting.integrity`). With no path it checks the shipped manifest.

## 9. Security / discipline preserved

All Phase 2/3/3.1/3.2 rules hold: pieces/recipes allowlisted + hash-verified; route-fact selection only; shadow_only
fails closed in live; unknown slots / brace-injection / unresolved placeholders fail closed; deterministic + auditable.
Added: provenance is `extra='forbid'`/frozen; provenance shadow-only fail-closed; integrity never renders a prompt.
No prompt wording / model / temperature / confidence / artifact copy change. No buyer-facing UI. No protocol-as-execution.

## 10. Tests and results

- `pytest tests/test_prompt_provenance.py tests/test_manifest_integrity.py -q` -> **41 passed**.
- `pytest tests/test_prompt_registry_contract.py tests/test_prompt_assembly_engine.py tests/test_mlb_prompt_equivalence.py tests/test_mlb_branch_overlays.py -q` (Phase 2/3/3.1/3.2) -> **60 passed** (unchanged).
- `pytest -q` (full agent-service suite) -> **227 passed, 0 failed** (186 prior + 41 new).
- `python scripts/check_prompt_manifest.py` -> `OK ... (8 templates, 9 recipes)`, exit 0.
- 9-regime byte-equivalence remains green; `manifest.json` + template files are unchanged (git-confirmed), so prompt
  bytes are byte-identical to Phase 3.2. `.NET` unchanged (no dotnet test needed).

## 11. Live analyzer unchanged confirmation

`git status` shows `services/agent-service/app/services/sports_analyzer.py`, `platform/` (.NET), and
`app/prompting/templates/` all UNCHANGED. Provenance/integrity are imported by nothing on the request path.

## 12. Remaining gaps

- **Parallel rule sets.** `integrity.verify_manifest_integrity` and `manifest.load_manifest` independently encode "a
  valid manifest." Consolidating them was deliberately deferred: `load_manifest` raises typed exceptions
  (`TemplateHashError`/`ManifestValidationError`) that Phase 2 tests assert against, so routing it through the
  fail-collecting verifier would not be behavior-preserving. Future consolidation should share a single rule-iterator
  that both a fail-fast and a fail-collecting caller consume.
- No provenance persistence sink yet (sec 7).
- The `.NET` `sourceDepth -> dataRegime` route-context computation is still a later slice (route facts are computed in
  .NET; provenance currently consumes fixture-supplied facts).
- Integrity does not yet run in an actual CI pipeline (the script exists; wiring it into CI is a follow-up).
- MLB only; other sports not modeled.

## 13. Recommended next slice

**Phase 4.1 -- Prompt Provenance Persistence v1 (still non-live):** choose/define a correct shadow-metadata audit sink
and persist the provenance record for a shadow assembly (additive, no live routing, no buyer surface). Separately: wire
`scripts/check_prompt_manifest.py` into CI, and the .NET `sourceDepth -> dataRegime` computation. Then live migration
(diff assembled vs live on exact equality behind a shadow flag) before any production routing. Defer protocol-as-execution.

## 14. Skill update recommendation

No skill file changed. The dai-specific slice skills referenced in `dai/CLAUDE.md` are not installed here; if/when they
are created, a reusable rule worth seeding: *provenance is metadata ABOUT an assembly and must be derived from the
already-rendered result, never by re-rendering -- so it is structurally incapable of changing prompt bytes; and a
manifest integrity check must be fail-collecting (report every defect) and separate from the fail-fast loader the
registry uses.*
