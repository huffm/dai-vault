# Data-Regime Overlay + MLB Prompt Equivalence v1

**date:** 2026-06-28
**status:** IMPLEMENTED (NON-LIVE) + VERIFIED. Adds overlay/recipe composition + a deterministic MLB dataRegime
mapping, and proves the non-live registry/builder path assembles **byte-identical** text to the live MLB prompt across
the four primary data regimes. The live prompt output is unchanged (the construction was extracted verbatim into a
behavior-preserving helper). No prompt wording/model/confidence tuning; no cohort work.
**type:** Phase 3.1 shadow-equivalence slice. dai-slice-runner + dai-skill-router gate + test-driven-development +
dai-test-discipline + systematic-debugging + verification-before-completion + dai-grill-with-vault.
**anchor:** prove the registry/builder can reproduce the live prompt before it goes anywhere near live routing.

See also: [[prompt-assembly-engine-v1]] (Phase 3), [[prompt-registry-contract-v1]] (Phase 2),
[[controlled-dynamic-prompt-assembly-architecture-v1]] (Phase 1).

## 1. Purpose

Model the conditional MLB prompt construction (starter present/absent x market present/absent) as composable overlays
selected by a typed dataRegime, and prove the assembled text equals the live `analyze_mlb` prompt for fixed fixtures.

## 2. Relationship to Prompt Assembly Engine v1

Phase 3 assembled one template. Phase 3.1 adds **recipe composition**: a recipe lists ordered template pieces (base +
overlays); the builder renders each piece and joins them deterministically. This is the minimal step toward the
architecture's layered overlay stack.

## 3. Data-regime mapping

`app/prompting/dataregime.py` -- `mlb_data_regime(starter_enriched: bool, market_backed: bool) -> str`, deterministic
from typed evidence (not model judgment). The four v1 regimes: `starter_enriched_market_backed`,
`starter_enriched_market_missing`, `starter_missing_market_backed`, `starter_missing_market_missing`.
(`starter_enriched` = a starter context with season-form quality is present; `market_backed` = a run-line market
context is present.)

## 4. Overlay / recipe design

- Pieces (all `shadow_only`, hash-verified): `mlb.pregame.base.v1` (matchup line), `mlb.overlay.starter.enriched.v1`,
  `mlb.overlay.starter.missing.v1`, `mlb.overlay.market.backed.v1`, `mlb.overlay.market.missing.v1`. Pieces have empty
  `requiredRouteFacts` so they are never directly route-selectable -- they compose only via a recipe.
- Recipes (4, in `manifest.recipes`): each maps a `dataRegime` route fact to an ordered `pieceIds` list
  (base + starter overlay + market overlay).
- Composition: the builder renders each piece, `rstrip("\n")`s it, and joins with `"\n\n"`. This reproduces the live
  `"\n".join([matchup, starter_section, market_section])` exactly (the live sections carry a leading newline; pieces
  store no leading newline and the `"\n\n"` join supplies the blank line).

## 5. MLB prompt equivalence method

The live MLB prompt construction was extracted verbatim from `analyze_mlb` into a pure helper
`build_mlb_user_message(home, away, date, starter_context, market_context)` in `sports_analyzer.py`. `analyze_mlb`
now calls it -- **byte-identical output, no behavior change** (confirmed by 109 existing analyzer tests + the
equivalence tests). The helper is the equivalence oracle: tests assert
`builder.render_recipe(...) == build_mlb_user_message(...)` for each regime fixture.

## 6. Fixture regimes

All four covered with a fixed fixture (Astros @ Tigers, 2026-06-28): enriched starters (both season forms present) and
a run-line market (`bookCount=1`, no multi-book depth). Each regime's recipe assembles byte-identically to the live
prompt for its fixture.

## 7. Hash / metadata behavior

`PromptAssemblyResult` (additive): `recipeId`, `dataRegime`, `pieceHashes` (per-piece SHA-256). For a recipe,
`templateHash` is empty (a recipe has no single body); `assembledHash` is SHA-256 of the final composed text.
`pieceHashes` are independent of slot values; `assembledHash` changes when any evidence slot changes.

## 8. Security rules preserved

All Phase 2/3 rules hold: pieces/recipes allowlisted + hash-verified; deterministic route-fact selection only; no
model/frontend prompt selection (route context still `extra='forbid'`); unknown slots (not in any recipe piece) fail
closed; brace-injecting slot values fail closed; no unresolved placeholders; shadow_only recipes/templates fail closed
in live mode; no secrets; no raw provider dumps. Recipe selection is deterministic and fail-closed (no match,
lifecycle-excluded, or ambiguous all -> fail closed).

## 9. Tests and results

`tests/test_mlb_prompt_equivalence.py` (17): dataRegime mapping (4 + completeness), byte-equivalence across the 4
regimes (parametrized), stable assembledHash, slot-change -> hash change, pieceHashes slot-independence + empty
templateHash, unknown-regime fail closed, brace-injection fail closed, unknown-slot fail closed, shadow-in-live fail
closed, assemble_recipe metadata completeness.

- `pytest tests/test_prompt_registry_contract.py tests/test_prompt_assembly_engine.py tests/test_mlb_prompt_equivalence.py -q` -> **47 passed**.
- `pytest -q` (full agent-service suite) -> **173 passed, 0 failed**. `.NET` unchanged (no dotnet test).

## 10. Live analyzer unchanged confirmation

The live **prompt output** is unchanged: `analyze_mlb` produces byte-identical user messages (verbatim extraction;
109 analyzer tests + 4 equivalence tests confirm). The registry/builder/overlays are imported by **nothing** on the
request path; all recipes/templates are `shadow_only` and fail closed in live mode. `sports_analyzer.py` was edited
only to extract `build_mlb_user_message` (a behavior-preserving refactor explicitly permitted by the slice).

## 11. Remaining gaps

- The `market_backed` overlay covers the run-line (no multi-book depth) variant only; the multi-book **depth
  sub-block** (consensus/median/disagreement/total) is a deferred overlay variant (`market_backed_depth`).
- The **starter-name-only** state (starter present, no season-form quality) is a deferred starter overlay
  (`starter_named`) -- v1 covers enriched and missing.
- `dataRegime` facts are still supplied by tests/fixtures; .NET route-context computation of `sourceDepth`/`dataRegime`
  is a later slice.
- No CI hash-check yet (manifest hashes vs files).

## 12. Recommended next slice

**Phase 3.2 -- Market Depth + Starter-Named Overlays v1:** add the deferred `market_backed_depth` and `starter_named`
overlays + recipes with byte-equivalence tests against the live depth/name-only branches (shadow-only). Then Phase 4
(prompt audit logging) and the .NET dataRegime computation. Defer protocol-as-execution. Add a CI hash-check.

## Skill update recommendation

No skill file was changed. The reusable lesson -- *prove shadow byte-equivalence to the live prompt (via a
behavior-preserving extraction used as the oracle) before any live prompt-routing migration* -- is project-specific
prompt-architecture doctrine and lives in this vault note + [[prompt-assembly-engine-v1]]. If a dedicated
prompt-architecture skill is later created (there is none today), this rule and the "controlled assembly, not
free-form generation" boundary should seed it. Updating a general slice/handoff skill for this single technique would
be out of scope.
