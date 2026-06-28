# Prompt Provenance Persistence v1

**date:** 2026-06-28
**status:** IMPLEMENTED (NON-LIVE) + VERIFIED. Adds (a) a narrow, append-only, shadow-only provenance audit sink
(`provenance_sink.py`) that durably captures a built `PromptProvenance` record, (b) an optional `sink` on
`PromptBuilder.assemble_recipe_with_provenance` so capture happens strictly downstream of assembly, and (c) the manifest
integrity check wired into the test surface as the actual CI command. NOT wired into the live analyzer; `sports_analyzer.py`
and `.NET` unchanged. No prompt/model/confidence tuning; no cohort work. DB persistence deliberately deferred (not yet
justified) with the exact future contract documented here.
**type:** Phase 4.1 foundation slice. test-driven-development + verification-before-completion + code-review (high). The
dai-specific skills referenced in `dai/CLAUDE.md` (dai-skill-router / dai-slice-runner / dai-grill-with-vault /
dai-docs-architect / dai-agent-handoff) are NOT installed in this environment; superpowers equivalents + a manual
structured handoff were substituted (recorded in the Skills Gate note of the handoff).
**anchor:** make a shadow recipe assembly's provenance DURABLE through the narrowest safe sink (append-only JSONL audit
file) without choosing a premature DB schema, keep capture downstream of assembly so prompt bytes stay byte-identical, and
keep manifest integrity runnable as one CI command -- all still shadow-only / non-live.

See also: [[prompt-provenance-and-manifest-integrity-v1]] (Phase 4), [[market-depth-starter-named-overlays-v1]] (Phase 3.2),
[[data-regime-overlay-mlb-prompt-equivalence-v1]] (Phase 3.1), [[prompt-assembly-engine-v1]] (Phase 3),
[[prompt-registry-contract-v1]] (Phase 2), [[controlled-dynamic-prompt-assembly-architecture-v1]] (Phase 1).

## 1. Purpose

Phase 4 returned a `PromptProvenance` record in-memory and explicitly deferred persistence because no clearly-correct sink
existed. Phase 4.1 chooses and implements the first safe persistence path, wires manifest integrity into the test/CI
surface, and assesses the .NET `sourceDepth -> dataRegime` mapping (design + defer). Everything stays non-live.

## 2. Persistence decision (sink chosen + why)

**Chosen sink: an append-only local JSONL audit file (`JsonlProvenanceSink`), plus an `InMemoryProvenanceSink` for tests,
behind a small `ProvenanceSink` Protocol. DB persistence is deferred (not yet justified).**

Reasoning, against the slice's decision rules:
- **Prefer the narrowest safe sink.** The agent service has NO database layer of its own (verified: no sqlite / sqlalchemy
  / pyodbc / engine anywhere under `services/agent-service/app`). The .NET platform owns persistence. The narrowest sink
  that is durable and adds zero runtime coupling is an append-only file the shadow path can write.
- **Do not create a broad schema unless the existing persistence model clearly supports it.** Routing provenance into the
  .NET DB now would force a premature `prompt_provenance` schema AND a cross-stack runtime coupling (the python shadow path
  would have to call the platform) -- both forbidden this slice. So no DB schema was created.
- **Append-only / audit-style.** `JsonlProvenanceSink.record` opens the file in `"a"` mode and writes exactly one JSON line
  per record; existing lines are never rewritten. One record = one line.
- **Downstream of assembly; never re-render.** A sink only accepts an already-built `PromptProvenance` object. It never
  reads slots, selects a recipe, or renders text -- so it is structurally incapable of changing prompt bytes.
- **Fail closed / fail loud, never affect live.** Capture is shadow-only at three layers (see sec 4). The builder method
  still raises before any capture for `mode != "shadow"`, so a live record can never reach a sink.

## 3. What was implemented (all in `dai`, non-live)

- `app/prompting/provenance_sink.py` (NEW):
  - `ProvenanceSink` (`typing.Protocol`, `runtime_checkable`) -- `record(provenance) -> None`.
  - `InMemoryProvenanceSink` -- non-durable, keeps `.records` in arrival order (tests/inspection).
  - `JsonlProvenanceSink(path)` -- durable append-only JSONL; `record()` writes `model_dump_json()` + newline (creates the
    parent dir on demand); `read_all()` parses every line back via the symmetric `PromptProvenance.model_validate_json`.
  - `capture_provenance(sink, provenance, *, mode: RouteMode)` -- shadow-only; fail-closed (`ProvenanceSinkError`) on a
    non-shadow `mode` arg AND on a record whose own `mode != "shadow"`.
  - `ProvenanceSinkError`.
- `app/prompting/builder.py` -- `assemble_recipe_with_provenance(..., sink: ProvenanceSink | None = None)`. When `sink` is
  given, the built record is captured via `capture_provenance` AFTER assembly. `sink=None` is byte-identical to Phase 4.
- `app/prompting/__init__.py` -- exports `ProvenanceSink`, `ProvenanceSinkError`, `InMemoryProvenanceSink`,
  `JsonlProvenanceSink`, `capture_provenance`.
- `tests/test_provenance_sink.py` (NEW, 17 cases incl. 9-regime parametrize).
- `tests/test_manifest_integrity.py` -- added test 13: runs the real CI wrapper `scripts/check_prompt_manifest.py` as a
  subprocess and asserts exit 0 + `OK` output (covers the command an operator/CI actually invokes, not just `main()`).

No `provenance.py` / `integrity.py` / `dataregime.py` / `manifest.json` / template / `.NET` change.

## 4. Shadow-only is enforced at three layers (intentional defense in depth)

1. `PromptBuilder.assemble_recipe_with_provenance` raises `PromptAssemblyError` for `mode != "shadow"` BEFORE building or
   capturing anything (fail fast; nothing is persisted on the live path).
2. `capture_provenance` refuses a non-shadow `mode` arg AND cross-checks `provenance.mode` (guards direct callers and the
   `InMemoryProvenanceSink`, which has no guard of its own).
3. `JsonlProvenanceSink.record` itself refuses any record whose `mode != "shadow"` (guards a direct `sink.record(...)`
   call that bypasses `capture_provenance`).

The overlap between (2) and (3) on a single call path is deliberate: deleting layer (3) would let a direct
`sink.record(live_record)` persist a live record to the durable audit file. Test 8 (`test_jsonl_sink_refuses_non_shadow_record`)
locks this in.

## 5. Minimum captured fields (persisted)

Every persisted line is a full `PromptProvenance`: `recipeId`, `recipeVersion`, `dataRegime`, `starterState`,
`marketState`, `pieceIds`, `pieceHashes`, `assembledHash`, `routeContext`, `outputSchemaId`, `lifecycle`, `mode`,
`createdUtc`. `read_all()` round-trips them back to typed records (pydantic coerces JSON arrays back to the tuple fields;
frozen-model `==` confirms the read record equals the original).

## 6. Future DB contract (deferred, exact)

When a DB sink becomes justified (i.e. when provenance must be queried across runs/tenants), implement a new
`ProvenanceSink` whose `record` writes one row to an **append-only** table `prompt_provenance`:
- columns = the `PromptProvenance` fields above, with `pieceIds`/`pieceHashes`/`routeContext` stored as JSON, plus a
  **non-null `tenantId`** and a surrogate `id`;
- natural audit key `(tenantId, createdUtc, assembledHash)`; rows are **never updated or deleted**;
- written **only** from shadow assembly (the same shadow-only gate); queried **tenant-scoped** (every read filters by
  `tenantId`).

This is a sink swap, not a contract change: the builder/`capture_provenance` path is unchanged. `tenantId` is the one field
not in `PromptProvenance` today (the route context carries `product`/`workflow`/`sport`/`competition` but not a tenant id);
the DB slice must thread a tenant id into capture before writing rows.

## 7. CI integrity status

- **Wired into the test surface:** test 13 executes `scripts/check_prompt_manifest.py` end-to-end and asserts exit 0, so a
  manifest regression fails the suite. Underlying `main()` exit codes + the real shipped manifest were already covered by
  Phase 4 tests 11-12.
- **The single CI command (from `services/agent-service`):** `python scripts/check_prompt_manifest.py` (exit 0 clean / 1
  defective; no path = the shipped manifest).
- **Actual CI pipeline wiring deferred (documented, not speculative-committed):** `dai/.github/workflows/` exists but is
  empty and untracked -- there is no tracked CI config to safely amend, and the monorepo also has a .NET side, so authoring
  a GitHub Actions workflow now (runner, python setup, dependency install) would be speculative. Ready-to-adopt step when CI
  is established:
  ```yaml
  - name: prompt manifest integrity
    working-directory: services/agent-service
    run: python scripts/check_prompt_manifest.py
  ```

## 8. .NET sourceDepth -> dataRegime (design + defer)

Assessed read-only. The mapping is well-defined and pure:

| .NET `SourceDepthRecord` | python state | regime token |
|---|---|---|
| `starting_pitching` = `none` (or no record) | `starter` = `missing` | `starter_missing_...` |
| `starting_pitching` = `identity_only` | `starter` = `named` | `starter_named_...` |
| `starting_pitching` = `enriched` | `starter` = `enriched` | `starter_enriched_...` |
| `market_odds` absent (no record) | `market` = `missing` | `..._market_missing` |
| `market_odds` = `shallow` | `market` = `backed` | `..._market_backed` |
| `market_odds` = `enriched` | `market` = `backed_depth` | `..._market_backed_depth` |

`dataRegime = "starter_{starterState}_market_{marketState}"` -- exactly what python `dataregime.mlb_regime` already owns.

**Decision: defer the .NET implementation.** Reasons: (1) the canonical regime mapping already lives in python
`dataregime.py`; emitting it independently in .NET creates two sources of truth that must be kept in sync -- ownership
should be decided before writing it. (2) Prompt routing is non-live, so an unwired .NET mapper would be dead code in the
live platform until something consumes it, and wiring it is the runtime-coupling step the slice forbids. (3) The slice
explicitly permits documenting and deferring. `.NET` is left untouched (git-confirmed). When the live migration slice
arrives, it should decide regime ownership (python canonical, .NET passes typed source-depth facts) and add the .NET mapping
+ tests then, behind the shadow flag.

## 9. Provenance still does not alter rendered prompt bytes (proven)

Capture is downstream of assembly and never re-renders. `test_provenance_sink.test_persistence_preserves_byte_equivalence`
asserts, for ALL 9 MLB regimes, that `render_recipe(...) == build_mlb_user_message(...)` (live oracle) byte-for-byte AND
that the persisted record's `assembledHash` equals the assembly result hash. Phase 4's byte-equivalence tests and
`test_mlb_prompt_equivalence` remain green. `mode="live"` fails closed before any capture.

## 10. Security / discipline preserved

All Phase 2/3/3.1/3.2/4 rules hold. Added: persistence is shadow-only at three layers and append-only; the sink never
re-renders; `capture_provenance` is typed `RouteMode`. No prompt wording / model / temperature / confidence / artifact copy
change. No live routing. No buyer-facing UI. No protocol-as-execution. No cohort settlement/capture. No Drive/FIFA. No
hidden background services.

## 11. Tests and results

- `pytest tests/test_provenance_sink.py -q` -> **17 passed**.
- `pytest tests/test_manifest_integrity.py -q` -> **13 passed** (12 prior + 1 CI-command test).
- `pytest tests/test_prompt_provenance.py tests/test_provenance_sink.py tests/test_manifest_integrity.py tests/test_mlb_prompt_equivalence.py tests/test_mlb_branch_overlays.py tests/test_prompt_assembly_engine.py tests/test_prompt_registry_contract.py -q` -> **119 passed**.
- `pytest -q` (full agent-service suite) -> **245 passed, 0 failed** (227 prior + 17 sink + 1 CI test).
- `python scripts/check_prompt_manifest.py` -> `OK ... (8 templates, 9 recipes)`, exit 0.
- `manifest.json` + template files unchanged (git-confirmed) -> prompt bytes byte-identical to Phase 3.2/4. `.NET` unchanged.

## 12. Code review

`/code-review high` (8 finder angles + verify). Resolved: (a) `read_all` now uses the symmetric
`PromptProvenance.model_validate_json` instead of manual `json.loads` kwargs construction (drops `import json`, honors any
future serializers/aliases); (b) `capture_provenance`'s `mode` is typed `RouteMode` (was bare `str`) so typos surface at
type-check time; (c) added a single-writer note to `JsonlProvenanceSink` (no cross-process lock in v1 -- concurrent writers
out of scope for the non-live shadow phase). Confirmed-and-kept: the three-layer shadow-only check is intentional defense in
depth (a verifier confirmed test 8 exercises the direct `sink.record` path). No correctness bugs survived verification.

## 13. Live analyzer unchanged confirmation

`git status` (before commit) showed only `app/prompting/__init__.py`, `app/prompting/builder.py`,
`tests/test_manifest_integrity.py` modified plus the two new files (`provenance_sink.py`, `test_provenance_sink.py`).
`sports_analyzer.py`, the FastAPI analyze path, `platform/` (.NET), `manifest.json`, and the templates are all UNCHANGED.
Nothing on the request path imports the sink.

## 14. Remaining gaps

- DB persistence not implemented (deliberate; exact contract in sec 6). The local JSONL sink has no cross-process lock
  (single-writer assumption).
- `tenantId` is not yet a provenance field; the DB slice must thread it in before writing tenant-scoped rows.
- `.NET sourceDepth -> dataRegime` mapping is designed but not implemented (sec 8).
- Integrity runs in the test suite + as a documented CI command, but is not yet in an actual CI pipeline (the workflows dir
  is empty/untracked).
- Live migration (diff assembled vs live on exact equality behind a shadow flag) is still pending. MLB only.

## 15. Recommended next slice

**Phase 4.2 -- Provenance DB Sink + Tenant Threading v1 (still non-live):** when querying provenance across runs is needed,
implement a `ProvenanceSink` backed by the append-only `prompt_provenance` table per sec 6, thread a `tenantId` into
capture, and keep the shadow-only gate. Alternatively (smaller, also non-live): **Live Migration Readiness v1** -- a shadow
flag that assembles via the registry alongside the live path and asserts exact byte-equality on real runs, emitting
provenance to the JSONL sink, before any production routing. Establish CI first if/when a pipeline is added, then wire
`scripts/check_prompt_manifest.py`. Defer the .NET mapping until the migration slice decides regime ownership. Defer
protocol-as-execution.

## 16. Skill update recommendation

No skill file changed (dai-specific slice skills not installed here). Reusable rule worth seeding when they exist:
*persistence of an audit record must be a downstream sink that accepts an already-built record (never re-derives or
re-renders), gated shadow-only at the boundary AND defensively in the durable sink; and when the local service has no DB,
prefer the narrowest durable sink (append-only file) + a documented future DB contract over forcing a premature schema or
cross-stack runtime coupling.*
