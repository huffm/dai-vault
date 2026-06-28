# Live Migration Readiness v1

**date:** 2026-06-28
**status:** IMPLEMENTED (NON-LIVE) + VERIFIED. Adds a validation harness that, from the SAME real analyzer input objects the
live path uses, derives the shadow route + slots, builds the live prompt and the shadow registry prompt, asserts byte
equality, and captures provenance through the existing `ProvenanceSink` ONLY after equality is proven. NOT live routing, NOT
DB persistence, NOT prompt tuning. `sports_analyzer.py` and `.NET` unchanged.
**type:** Live Migration Readiness v1 slice. test-driven-development + verification-before-completion + code-review (high).
The dai-specific slice skills referenced in `dai/CLAUDE.md` (dai-skill-router / dai-slice-runner / dai-grill-with-vault /
dai-docs-architect / dai-agent-handoff) are NOT installed in this environment; superpowers equivalents + a manual structured
handoff were substituted (recorded in the Skills Gate note of the handoff).
**anchor:** prove the non-live prompt registry can reproduce the live MLB prompt byte-for-byte FROM REAL ANALYZER INPUTS
(not hand-built fixtures), and capture provenance only once equality is proven -- the last readiness gate before any
production routing. Still shadow-only; live generation stays authoritative; prompt bytes byte-identical.

See also: [[prompt-provenance-persistence-v1]] (Phase 4.1), [[prompt-provenance-and-manifest-integrity-v1]] (Phase 4),
[[market-depth-starter-named-overlays-v1]] (Phase 3.2), [[data-regime-overlay-mlb-prompt-equivalence-v1]] (Phase 3.1),
[[prompt-assembly-engine-v1]] (Phase 3), [[prompt-registry-contract-v1]] (Phase 2),
[[controlled-dynamic-prompt-assembly-architecture-v1]] (Phase 1).

## 1. Purpose

Phases 3.1/3.2/4 proved shadow == live for hand-built slot fixtures. The migration-readiness gap was: can the shadow inputs
be DERIVED from the same real analyzer objects (`MlbStarterContext` / `BaseballMarketContext` / teams / date) the live path
consumes, and still reproduce the live bytes? This slice closes that gap with a harness + a pure mapper, and wires provenance
capture to fire only after byte equality is proven -- without touching the live analyzer prompt path.

## 2. Why DB persistence remains deferred

Unchanged from [[prompt-provenance-persistence-v1]]: the agent service has no DB layer (the .NET platform owns persistence),
so a DB sink would force a premature `prompt_provenance` schema + cross-stack runtime coupling -- both forbidden. The harness
captures through the existing `ProvenanceSink` abstraction, so it works with `InMemoryProvenanceSink` (tests) or the
append-only `JsonlProvenanceSink` (durable, local) and will work with a future DB sink unchanged (the documented future
contract in `prompt-provenance-persistence-v1.md` sec 6 still stands: append-only `prompt_provenance` table, non-null
`tenantId`, audit key `(tenantId, createdUtc, assembledHash)`, never updated/deleted, tenant-scoped reads).

## 3. How real-run shadow equality is validated

`app/services/migration_readiness.py` (NEW):
- `mlb_route_and_slots(home, away, date, starter_ctx, market_ctx) -> (PromptRouteContext, slots, regime)` -- PURE. Derives
  the regime via the same typed-evidence rules the live prompt branches on (`mlb_starter_state` / `mlb_market_state` /
  `mlb_regime`), and derives slot values with the SAME formatters the live prompt uses:
  - starter form via `_format_pitcher_quality` (imported from `sports_analyzer` -- deliberately the live formatter, so the
    form phrase cannot drift from live);
  - market-depth phrases with the same numeric specs (`{x*100:.0f}` for %, `{x:.1f}` for the total) and the same
    `< 0.05` "books largely agree" / else "books disagree noticeably" threshold.
- `check_mlb_shadow_equivalence(registry, builder, home, away, date, starter_ctx, market_ctx, *, sink=None, created_utc=None)`
  -- builds the live prompt (authoritative), composes the shadow prompt ONCE via
  `PromptBuilder.assemble_recipe_for_migration` (returns `(text, result, provenance)` from a single render), asserts
  `shadow == live`, and only then captures provenance.
- `PromptBuilder.assemble_recipe_for_migration` (NEW, in `builder.py`) -- shadow-only single-compose helper; provenance is
  built from that one render, never a re-render. `_build_recipe_result` now also surfaces the rendered `text` (so equality,
  hash, and provenance all come from one compose); its two existing callers were updated (behavior-preserving).

Equality is a literal byte comparison of the rendered text (`shadow == live`), proven for all 9 MLB regimes from derived
inputs.

## 4. When provenance is captured (and when it is not)

- **Captured** only when (a) a `sink` is given AND (b) `shadow == live` was proven first. Capture uses the provenance built
  from the SAME single render (downstream, never re-rendered), via the existing shadow-only `capture_provenance`.
- **Not captured** on any readiness gap, which fails loud in one of two ways and persists nothing:
  - `ShadowEqualityError` -- the shadow bytes differ from live.
  - `PromptAssemblyError` -- the shadow recipe cannot be assembled for the input shape (partial evidence: one starter has
    season quality and the other does not, or a multi-book market is missing one depth metric; the live prompt omits that
    line but the overlay template requires the slot, so assembly fails closed BEFORE the equality compare).
  A driver iterating real games should treat BOTH as "not ready" signals. Both are proven fail-loud-no-capture by tests.

## 5. Why live behavior is unchanged

`build_mlb_user_message` (and `_format_pitcher_quality`) are imported READ-ONLY; `sports_analyzer.py` is byte-for-byte
unchanged (git-confirmed). The harness is not wired into the FastAPI request path -- nothing in `routes/` or `analyze_*`
imports it. The shadow path stays shadow-only: `assemble_recipe_for_migration` and the recipes fail closed for `mode="live"`.
`manifest.json`, the templates, and `.NET` are all unchanged. So the live prompt bytes and the production request path are
behavior-preserving.

## 6. .NET sourceDepth -> dataRegime (still deferred)

Not required for this slice and not implemented (no read-only need arose). Regime ownership remains with PYTHON
(`dataregime.py` is the single source of truth; `mlb_route_and_slots` consumes it). The mapping table designed in
`prompt-provenance-persistence-v1.md` sec 8 still stands; the live-migration slice that eventually routes production prompts
should decide regime ownership (python canonical, .NET passes typed source-depth facts) and add the .NET mapping + tests
THEN, behind the shadow flag -- not as dead code now.

## 7. Files changed (all in `dai`, non-live)

- `app/services/migration_readiness.py` (NEW) -- `mlb_route_and_slots`, `check_mlb_shadow_equivalence`,
  `ShadowEqualityError`, `MigrationReadinessResult`, plus private state/slot derivation helpers.
- `app/prompting/builder.py` -- `assemble_recipe_for_migration` (NEW); `_build_recipe_result` returns the rendered text too
  (4-tuple); two existing callers updated.
- `tests/test_migration_readiness.py` (NEW, 17 cases incl. 9-regime parametrize + partial-evidence fail-loud).
- No change to `sports_analyzer.py`, `provenance.py`, `provenance_sink.py`, `integrity.py`, `dataregime.py`, `manifest.json`,
  templates, or `.NET`.

## 8. Exact tests run + results (venv python, from `services/agent-service`)

- `pytest tests/test_migration_readiness.py -q` -> **17 passed**.
- `pytest tests/test_mlb_prompt_equivalence.py tests/test_mlb_branch_overlays.py -q` (9-regime equivalence) -> **30 passed**
  (unchanged).
- `pytest -q` (full agent-service suite) -> **262 passed, 0 failed** (245 prior + 16 migration + 1 partial-evidence test).
- `python scripts/check_prompt_manifest.py` -> `OK ... (8 templates, 9 recipes)`, exit 0.
- `git status` confirms only `builder.py` modified plus the two new files; `sports_analyzer.py` / templates / `manifest.json`
  / `.NET` unchanged -> prompt bytes byte-identical.

## 9. Code review

`/code-review high` (8 finder angles + verify). Resolved: (a) documented BOTH fail-loud readiness modes
(`ShadowEqualityError` for byte divergence, `PromptAssemblyError` for partial-evidence input shapes) -- the docstring
previously claimed only the former; (b) added a partial-evidence test (one-sided starter quality) proving fail-loud +
no-capture; (c) removed the redundant `MigrationReadinessResult.equal` field (always True on return -- one unambiguous
failure channel via exception); (d) narrowed `MigrationReadinessResult.provenance` to non-Optional (always built). Verified
and kept (deliberate): reusing the live `_format_pitcher_quality` as the oracle's formatter (prevents drift -- duplicating it
would be worse); fully de-duplicating the market-depth phrase formatting would require extracting a shared helper from
`sports_analyzer.py`, which the slice forbids touching, so it is left as a test-guarded drift surface (sec 10). No correctness
bug survived verification.

## 10. Remaining risks / gaps

- **Market-depth phrase duplication.** The `< 0.05` threshold and the "books largely agree"/"books disagree noticeably"
  strings + numeric specs exist verbatim in both `sports_analyzer.build_mlb_user_message` and
  `migration_readiness._market_slots` (the live prompt builds them inline, with no reusable helper to import). This is a drift
  surface, but a TEST-GUARDED one: the 9-regime equivalence + harness tests fail loud if the two ever diverge. The clean fix
  (extract one shared depth-phrase formatter consumed by both) requires editing `sports_analyzer.py` and belongs to the live
  migration slice. Starter-form formatting has no such duplication (the harness reuses the live `_format_pitcher_quality`).
- **Private-symbol dependency.** The harness imports the `_`-prefixed `_format_pitcher_quality`. Intentional (fidelity), but
  it depends on a function with no stability contract; the migration slice that touches `sports_analyzer.py` should promote
  it to a public name.
- **Partial-evidence input shapes** (one-sided starter quality, partial multi-book depth) are real production shapes that the
  current overlay templates cannot represent; the harness correctly fails loud (PromptAssemblyError) and captures nothing,
  but the templates do not yet cover them. A later overlay slice could add partial-evidence variants if those shapes prove
  common.
- No DB persistence (deliberate; sec 2). The local JSONL sink is single-writer. MLB only. Still non-live -- no production
  routing yet.

## 11. Next recommended slice

**Live Prompt Routing (Shadow-Parallel) v1:** run `check_mlb_shadow_equivalence` INSIDE the real analyzer request path behind
an explicit, default-off shadow-validation flag -- assemble the shadow prompt alongside the authoritative live prompt on real
runs, assert byte equality, emit provenance to the JSONL sink, and (crucially) prove with regression tests that the flag
DISABLED leaves the request path byte-identical. Before that flag controls anything, extract the shared MLB depth-phrase
formatter (the one `sports_analyzer.py` edit this slice deferred) so the oracle and the live path share one source of truth,
and promote `_format_pitcher_quality` to public. Only after a clean shadow-parallel run across a real cohort should live
routing flip. Defer the .NET `sourceDepth -> dataRegime` mapping until that slice decides regime ownership. Defer DB
persistence, protocol-as-execution, and buyer UI.
