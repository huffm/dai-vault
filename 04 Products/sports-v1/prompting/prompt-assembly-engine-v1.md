# Prompt Assembly Engine v1

**date:** 2026-06-28
**status:** IMPLEMENTED (NON-LIVE) + VERIFIED. Adds a `PromptBuilder` that assembles a prompt from an approved,
hash-verified template and typed slot values, returning a deterministic `PromptAssemblyResult` (template + assembled
hashes, route context, output schema, slot metadata, reason). NOT wired into the live analyzer; `sports_analyzer.py`
unchanged. No prompt/model/confidence tuning; no cohort work.
**type:** Phase 3 foundation slice. dai-slice-runner + dai-skill-router gate + test-driven-development +
dai-test-discipline + systematic-debugging + verification-before-completion + dai-grill-with-vault.
**anchor:** turn the Phase 2 selected template into a safely assembled prompt -- controlled slot rendering, fail
closed, deterministic, hashable -- still shadow-only.

See also: [[prompt-registry-contract-v1]] (Phase 2), [[controlled-dynamic-prompt-assembly-architecture-v1]] (Phase 1).

## 1. Purpose

Provide the Builder layer: given a `PromptRouteContext`, a selected `PromptTemplate`, and typed slot values, produce a
deterministic assembled prompt + metadata, with fail-closed validation and no free-form generation.

## 2. Relationship to Prompt Registry Contract v1

Phase 2 added the contracts + manifest/hash verification + deterministic route selection. Phase 3 consumes a selected
template and **assembles** it. Live migration (routing production calls through the registry) is still future.

## 3. What was implemented (all in `dai`, non-live)

- `app/prompting/builder.py` -- `PromptBuilder(templates_dir)` with `assemble(template, route_context, slot_values)`
  and `assemble_selected(registry, route_context, slot_values, mode)`; `PromptAssemblyError` (fail-closed).
- `app/prompting/contracts.py` -- `PromptAssemblyResult` gains `unresolvedSlots` (always empty on success);
  `assembledHash` + `providedSlots` are now populated by the builder.
- `app/prompting/__init__.py` -- exports `PromptBuilder`, `PromptAssemblyError`.
- Reuses the Phase 2 `shadow_only` template `mlb.pregame.analysis.current_equivalent.v1` unchanged (no manifest/hash
  churn; Phase 2 tests stay green).
- `tests/test_prompt_assembly_engine.py` (13 tests, incl. a golden assembled hash).

## 4. What remains non-live

`sports_analyzer.py` and the FastAPI analyze path are unchanged (git-confirmed). Nothing on the request path imports
the builder. The example template is `shadow_only`; `assemble_selected(..., mode="live")` fails closed. No `.NET` change.

## 5. PromptBuilder behavior

`assemble`: validates slots, re-verifies the template file hash (defense in depth), renders, hashes, returns a
`PromptAssemblyResult`. `assemble_selected`: runs `registry.select(ctx, mode)` first (lifecycle/mode gated), then
assembles; fails closed if no single template is selectable (e.g. shadow_only in live mode).

## 6. Slot rendering rules (controlled, fail closed)

- Placeholders are exactly `{{ name }}` (identifier names); nothing else is interpolated.
- Every `requiredSlots` entry must be provided (missing -> fail closed).
- Every provided slot must be in `requiredSlots` (allowed set = required in v1; unknown -> fail closed).
- Every placeholder in the template body must be a declared required slot (undeclared -> fail closed).
- A slot value may not contain `{{` or `}}` (no placeholder injection) and is length-capped (20000) -> fail closed.
- Single deterministic substitution pass; values are never re-scanned. Any leftover placeholder after render ->
  fail closed. (Unreachable in normal flow given the guards above, but enforced as a final invariant.)

## 7. Hash rules

- `templateHash` = the manifest SHA-256 of the template **file bytes** -- independent of slot values.
- `assembledHash` = SHA-256 of the **final assembled prompt text** (utf-8).
- Tests prove: deterministic assembledHash for identical inputs; changing a slot changes assembledHash; templateHash
  is identical across different slot values; the two hashes are distinct concepts.

## 8. Assembly metadata (`PromptAssemblyResult`)

templateId, version, templateHash, assembledHash, routeContext, outputSchemaId, requiredSlots, providedSlots,
assemblyReason, lifecycle, unresolvedSlots. No DB persistence and no audit-log persistence (that is Phase 4).

## 9. Golden tests

Fixed shadow MLB slots (Astros @ Tigers, 2026-06-28, sample starter/market blocks) assemble to a stable golden
`assembledHash = 9bca5e1f...` (also reproduced independently in-test by substituting into the template file and
hashing). Verified: golden stability, slot-change -> hash change, templateHash slot-independence. Full byte-equivalence
to the *current live* `analyze_mlb` prompt is intentionally deferred (see sec 12) -- the live prompt has data-dependent
branches (no-data blocks, multi-book depth) that exceed a single fixed template; this slice proves the engine with a
representative shadow template and a golden fixture.

## 10. Security constraints (enforced)

No arbitrary prompt-text selector (`PromptRouteContext` is `extra='forbid'`); no unresolved placeholders; no hidden
slots (allowed = declared required); no unknown slot injection; no live use unless lifecycle allows; no slot value may
inject template braces; deterministic, hashable output; template file re-verified before render. No secrets in the test
template.

## 11. Live analyzer unchanged confirmation

`git status services/agent-service/app/services/sports_analyzer.py` = UNCHANGED. No production prompt/model/confidence
change. Builder imported by nothing on the request path; example template unselectable in live mode.

## 12. Migration path toward byte-equivalent MLB assembly

The live `analyze_mlb` prompt is **data-conditional** (starter present/absent, market present/absent, multi-book depth)
-- not a single static template. To reach byte-equivalence: (a) model the conditional blocks as data-regime overlay
fragments (e.g. `starter_present` vs `starter_missing`, `market_depth` vs `market_single`) selected by `sourceDepth`
facts; (b) add an `active` recipe per regime whose composed output equals the live string for that regime; (c) a
golden-equality test per regime against the live `analyze_mlb` output for a fixed fixture; (d) ship behind a shadow
flag, diffing assembled vs live, only on exact equality. This is a focused follow-up; this slice delivers the engine
and one golden, not the full live equivalence.

## 13. Risks / open questions

- Single-template assembly does not yet compose multiple layers/overlays (architecture's layered stack). The conditional
  MLB prompt needs overlay composition -- a Builder extension for the equivalence slice.
- `dataRegime` is still a free string; the canonical `dataRegime`-from-`sourceDepth` mapping remains a prerequisite for
  regime overlays.
- Whether the Builder/registry ultimately lives in Python or .NET is still open (route facts are computed in .NET).
- No CI hash-check yet (manifest vs file) -- recommended before live use.

## 14. Recommended next slice

**Phase 3.1 -- Data-Regime Overlay + MLB Prompt Equivalence v1:** add overlay composition + the `dataRegime`-from-
`sourceDepth` mapping, and a golden-equality test proving the assembled MLB prompt equals the current live
`analyze_mlb` output for fixed fixtures across regimes -- still shadow-only, no live change. Then Phase 4 (audit
logging). Defer protocol-as-execution.
