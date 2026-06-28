# Market Depth + Starter-Named Overlays v1

**date:** 2026-06-28
**status:** IMPLEMENTED (NON-LIVE) + VERIFIED. Closes the two deferred MLB prompt branches -- multi-book market depth
and starter named-only -- so the registry/builder recipe path is now byte-equivalent to the live MLB prompt across all
nine data regimes. Live prompt output unchanged; `sports_analyzer.py` NOT modified this slice. No prompt wording/model/
confidence tuning; no cohort work.
**type:** Phase 3.2 shadow-equivalence slice. dai-slice-runner + dai-skill-router gate + test-driven-development +
dai-test-discipline + systematic-debugging + verification-before-completion + dai-grill-with-vault.
**anchor:** branch-complete shadow byte-equivalence before any audit logging or live prompt migration.

See also: [[data-regime-overlay-mlb-prompt-equivalence-v1]] (Phase 3.1), [[prompt-assembly-engine-v1]] (Phase 3),
[[prompt-registry-contract-v1]] (Phase 2), [[controlled-dynamic-prompt-assembly-architecture-v1]] (Phase 1).

## 1. Purpose

Cover the remaining conditional branches of the live `build_mlb_user_message` (multi-book market depth; starter
names-only) with hash-verified overlays + recipes, proving byte-equivalence in shadow mode.

## 2. Relationship to Data-Regime Overlay + MLB Prompt Equivalence v1

Phase 3.1 covered 4 regimes (enriched/missing starters x backed/missing market). This slice adds the `named` starter
state and the `backed_depth` market state, completing the 3x3 state matrix (9 regimes).

## 3. Branches investigated

- **Market depth / multi-book detail** (live `build_mlb_user_message`): appended to the market section.
- **Starter named-only** (live starter block): names + handedness present, no season-form quality.

## 4. Live prompt conditions found (from the oracle)

- **Market depth** appears when `(market_context.bookCount or 0) >= 2 AND market_context.consensusSide`. It then
  appends, directly after the run-line lines (single `\n`, not a blank line): a `[market depth: N books surveyed]`
  line, a consensus line with optional space-joined sub-clauses -- median implied prob (when `medianHomeImpliedProb`),
  across-book disagreement (when `disagreementRange`; strength = "books largely agree" if `<0.05` else "books disagree
  noticeably"), median total (when `medianTotalPoint`) -- and a closing "weigh how firmly..." line. Because depth is
  concatenated into the SAME section, it is modeled as a single overlay piece (run-line + depth), not two pieces.
- **Starter named-only**: when `starter_context` is present but `_format_pitcher_quality` returns None for both
  starters, the block omits both season-form lines and ends with "do not fabricate era or recent form statistics --
  only name and handedness are available." (the else branch).

## 5. Overlay / recipe additions

New shadow_only pieces: `mlb.overlay.starter.named.v1` (names+handedness, no form) and
`mlb.overlay.market.backed_depth.v1` (run-line + full multi-book depth, all sub-clauses present). Five new recipes:
`starter_named_market_backed`, `starter_named_market_missing`, `starter_named_market_backed_depth`,
`starter_enriched_market_backed_depth`, `starter_missing_market_backed_depth`. All pieces hash-verified; pieces keep
empty `requiredRouteFacts` (not directly route-selectable); composition unchanged (`\n\n` join of rstripped pieces).

## 6. DataRegime / state mapping

`dataregime.py` (additive, backward-compatible): state dimensions `STARTER_STATES = (enriched, named, missing)`,
`MARKET_STATES = (backed, backed_depth, missing)`; `mlb_regime(starter_state, market_state) ->
"starter_{s}_market_{m}"`. Typed detectors: `mlb_starter_state(present, quality_present)`,
`mlb_market_state(present, depth_present)` (depth mirrors the live `bookCount>=2 and consensusSide` condition). The
Phase 3.1 boolean `mlb_data_regime(starter_enriched, market_backed)` is preserved as a facade over `mlb_regime`.
`PromptRecipe` gains optional `starterState`/`marketState` metadata (auditable; dataRegime still encodes both).

## 7. Byte-equivalence method

Same oracle as Phase 3.1: `render_recipe(...) == build_mlb_user_message(...)` byte-for-byte. Depth leaf values
(median %, disagreement %, total, strength label) are formatted in the fixture exactly as the live code formats them
(`*100:.0f`, `:.1f`, the `<0.05` strength branch) and passed as typed slots; the overlay carries the structure.

## 8. Fixture coverage

Astros @ Tigers, 2026-06-28. New branches proven byte-identical: starter named-only + backed; named + missing; named +
backed_depth; enriched + backed_depth; missing + backed_depth. Combined with Phase 3.1, all 9 regimes are covered.
Depth fixture has all sub-clauses present (median + disagreement + total); partial-depth permutations (e.g. median
only) are a deferred fixture set (the structure is the same overlay with fewer leaf clauses present).

## 9. Hash / metadata behavior

Unchanged from Phase 3.1: per-piece `pieceHashes` (slot-independent), `assembledHash` over final text (changes on any
branch slot, e.g. `median_home_prob`), recipe `templateHash` empty. `dataRegime` + new `starterState`/`marketState`
are on the recipe metadata.

## 10. Security rules preserved

All Phase 2/3/3.1 rules hold: pieces/recipes allowlisted + hash-verified; route-fact selection only; no model/frontend
prompt selection; unknown slots / brace-injection / unresolved placeholders fail closed; shadow_only fail closed in
live mode; deterministic + auditable recipe reason; no secrets; no raw provider dumps.

## 11. Tests and results

`tests/test_mlb_branch_overlays.py` (13): starter/market state detection; byte-equivalence for the 5 new branches
(parametrized); branch hash stable + slot-sensitive (median change); pieceHashes slot-independent; unknown-state /
unknown-slot / brace-injection / shadow-in-live fail closed.

- `pytest tests/test_prompt_registry_contract.py tests/test_prompt_assembly_engine.py tests/test_mlb_prompt_equivalence.py tests/test_mlb_branch_overlays.py -q` -> **60 passed**.
- `pytest -q` (full agent-service suite) -> **186 passed, 0 failed**. `.NET` unchanged (no dotnet test).

## 12. Live analyzer unchanged confirmation

`sports_analyzer.py` was NOT modified this slice (git-confirmed) -- the `build_mlb_user_message` oracle already existed
from Phase 3.1. The registry/builder/overlays are imported by nothing on the request path; all recipes/templates are
shadow_only and fail closed in live mode. Live prompt output is unchanged.

## 13. Remaining gaps

- Partial-depth permutations (median-only, disagreement-only, etc.) are not separate fixtures -- the same depth overlay
  represents the all-clauses-present case; sub-clause omission would need either conditional slots or additional depth
  overlay variants. Deferred.
- `dataRegime`/`sourceDepth` facts are still supplied by fixtures; .NET route-context computation is a later slice.
- No CI hash-check yet (manifest hashes vs files).
- Other sports (NFL/NBA) and soccer are not modeled in the registry (MLB only).

## 14. Recommended next slice

**Phase 4 -- Prompt Audit Logging v1 (still non-live):** persist the assembly metadata (recipeId/version/dataRegime/
pieceHashes/assembledHash/routeContext/outputSchemaId) for a run when computed in shadow, so prompt provenance is
auditable. Separately: a CI hash-check (manifest vs files), and the .NET `sourceDepth`->`dataRegime` computation,
before any live migration. Defer protocol-as-execution.

## 15. Skill update recommendation

No skill file changed (same rationale as Phase 3.1: no prompt-architecture skill exists today). Reusable rule worth
seeding such a skill if/when created: *a prompt migration must be branch-complete under shadow byte-equivalence
(every live conditional branch reproduced exactly, via a behavior-preserving extraction oracle) before audit logging
or live routing.* This doctrine lives in this note + the Phase 3/3.1 notes. Editing a general slice/handoff skill for
this single technique is out of scope.
