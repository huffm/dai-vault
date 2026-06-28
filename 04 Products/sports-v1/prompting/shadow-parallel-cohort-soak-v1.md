# Shadow-Parallel Cohort Soak v1

**date:** 2026-06-28
**status:** IMPLEMENTED + RUN + VERIFIED. An operational harness that runs a representative MLB cohort through the
default-off shadow validator (in enabled mode) and produces a go/no-go soak report. Soak result: **CLEAN / GO** -- 18/18
representable games matched the live prompt byte-for-byte across all 9 regimes and 18 provenance records were captured, with
zero mismatches / assembly failures / sink failures / errors. NON-LIVE: no model call, no second artifact, no betting outcome.
The live prompt path and its authority are untouched; `sports_analyzer.py` and `.NET` are unchanged this slice.
**type:** Shadow-Parallel Cohort Soak v1 slice. test-driven-development + verification-before-completion + code-review (high).
The dai-specific slice skills referenced in `dai/CLAUDE.md` (dai-skill-router / dai-slice-runner / dai-grill-with-vault /
dai-docs-architect / dai-agent-handoff) are NOT installed in this environment; superpowers equivalents + a manual structured
handoff were substituted (recorded in the Skills Gate note of the handoff).
**anchor:** produce the empirical evidence -- a clean shadow-parallel soak across all regimes with provenance captured -- that
a future registry-authoritative migration would require, WITHOUT enabling the validator globally, changing live prompt
authority, or making a model call.

See also: [[live-prompt-routing-shadow-parallel-v1]], [[live-migration-readiness-v1]], [[prompt-provenance-persistence-v1]]
(Phase 4.1), [[prompt-provenance-and-manifest-integrity-v1]] (Phase 4).

## 1. Cohort source and size

**Representative fixtures (NOT live data).** 18 games = all 9 MLB regimes (3 starter states x 3 market states) across 2
representative matchups (Tigers/Astros 2026-06-28, Yankees/Red Sox 2026-06-29). Every game is fully populated for its regime,
so the shadow recipe is representable and should reproduce the live prompt byte-for-byte.

- **Why representative:** they exercise every regime branch of `build_mlb_user_message` (starter enriched/named/missing x
  market backed/backed_depth/missing) with the exact slot-derivation the analyzer would use, so they prove the registry/oracle
  reproduces the live prompt bytes across the full branch matrix.
- **What they do NOT prove:** nothing about live retrieval coverage, real data quality/availability, model behavior, or betting
  outcomes. No live data was used (no DB/network access in this environment, and the slice forbids cohort settlement /
  calibration scoring). This soak is a PROMPT-EQUIVALENCE soak, not a data or accuracy soak.
- **Known un-representable shapes:** one-sided starter quality and partial multi-book depth are not representable by the
  current overlay templates; they are exercised by a separate edge probe and counted as
  `partial_evidence_unrepresentable` (a documented template gap, not drift).

## 2. Enablement method

The validator stays default-off. The soak enables it ONLY through an explicit `ShadowValidationConfig(enabled=True,
sink_path=...)` passed into `run_soak` -- never via global env, never a permanent config change. The harness calls the exact
sidecar the analyzer runs when enabled (`run_mlb_shadow_validation`), so the soak validates the real enabled-mode codepath.
It makes no model call (the sidecar is independent of the model response).

## 3. Sink path

A caller-provided JSONL path (append-only `JsonlProvenanceSink`). For the recorded run the sink was a scratchpad path OUTSIDE
the repo; the runnable script defaults to a temp path and freshly truncates it each run so the persisted-line count reflects
that run. **No JSONL artifact is committed.**

## 4. Summary counts (recorded run)

Command (from `services/agent-service`): `python scripts/run_shadow_cohort_soak.py <scratchpad>/shadow_soak_provenance.jsonl`

```
cohort_size:                       18
attempted:                         18
matched:                           18
captured:                          18
mismatched:                         0
assembly_failed:                    0
sink_failed:                        0
errored:                            0
skipped_no_sink:                    0
skipped_disabled:                   0
partial_evidence_unrepresentable:   0   (clean cohort has no partial-evidence games)
regimes:  all 9 regimes x2 (starter_{enriched,named,missing}_market_{backed,backed_depth,missing})
failures: []
clean:  true
go:     true   (clean AND matched == cohort_size)
persisted_provenance_lines: 18
exit code: 0
```

Provenance sanity (read back from the JSONL): 18 lines, all `mode == "shadow"`, all `lifecycle == "shadow_only"`, 18 distinct
`assembledHash` (64-char), 9 distinct `dataRegime`.

## 5. Any mismatches / failures

**None.** Zero mismatches, assembly failures, sink failures, or errors on the representable cohort. Per the slice's stop rule,
a clean soak means no failure shape was observed; nothing blocks documenting readiness for the next (still-gated) step.

## 6. Provenance capture status

All 18 representable games captured provenance to the JSONL sink, each AFTER byte equality was proven (capture is downstream
of the single shadow compose, never re-rendered). The edge probe (partial-evidence) captured nothing (fail-loud, no capture),
as designed.

## 7. Performance / operational observations

- The soak loads the registry + builds the builder once PER game (18x), deliberately: it exercises the exact per-call sidecar
  the analyzer runs in production, rather than a hoisted-cache variant. A module-level registry/builder cache remains the
  obvious optimization if the validator is ever run hot (documented in [[live-prompt-routing-shadow-parallel-v1]]).
- The full 18-game soak runs in well under a second with no model call. No network or DB access.
- The go/no-go gate requires `clean AND matched == cohort_size`, so a soak that silently skipped every game
  (e.g. a config-plumbing regression) is clean-but-empty and correctly does NOT pass (exit 1) -- broad-guard safety is never
  treated as success.

## 8. Why live behavior remains unchanged

`sports_analyzer.py` is unchanged this slice (git-confirmed). The harness only calls `run_mlb_shadow_validation` (the existing
default-off sidecar) and the existing oracle; it does not touch the live prompt, the model call, or buyer output. The only
modified request-path-adjacent file is `shadow_validation.py`, and its change is additive: a `reason` classifier on the
outcome + a refactor of the sidecar into "prove equality (no sink) then capture (isolated)" so a sink-write failure is
classified distinctly and still never breaks the live request. The existing shadow-validation tests (16) and the 9-regime
byte-equivalence tests remain green, so live prompt bytes are unchanged.

## 9. Files changed (all in `dai`)

- `app/services/shadow_validation.py` -- added `ShadowValidationOutcome.reason`; restructured
  `run_mlb_shadow_validation` into a step-1 equality check (sink=None) + step-2 isolated capture, so `sink_failed` is distinct
  and a sink error never masks/aliases drift. Behavior preserved for the analyzer enabled path.
- `app/services/shadow_cohort_soak.py` (NEW) -- `CohortGame`, `SoakSummary` (with derived `attempted` + `clean`),
  `build_representative_cohort` (18 games), `build_partial_evidence_probe`, `run_soak`, `_classify`.
- `scripts/run_shadow_cohort_soak.py` (NEW) -- runnable; fresh-truncates the sink, prints the json report, exits 0 iff GO.
- `tests/test_shadow_cohort_soak.py` (NEW, 9 cases).
- No change to `sports_analyzer.py`, the prompting package, `manifest.json`, templates, or `.NET`.

## 10. Exact tests run + results (venv python, from `services/agent-service`)

- `pytest tests/test_shadow_cohort_soak.py -q` -> **9 passed**.
- `pytest tests/test_shadow_validation.py -q` -> **16 passed** (unchanged behavior after the reason/refactor).
- `pytest tests/test_mlb_prompt_equivalence.py tests/test_mlb_branch_overlays.py tests/test_migration_readiness.py -q` ->
  green (byte-equivalence + migration unchanged).
- `pytest -q` (full agent-service suite) -> **287 passed, 0 failed** (278 prior + 9 new).
- `python scripts/check_prompt_manifest.py` -> `OK ... (8 templates, 9 recipes)`, exit 0.
- `python scripts/run_shadow_cohort_soak.py <scratchpad path>` -> GO, clean, 18/18 matched+captured, exit 0.

## 11. Code review

`/code-review high` (8 finder angles + verify). Resolved: (a) partial-evidence outcomes are no longer appended to the
`failures` list (a known non-drift gap was polluting a list named `failures`); (b) added a distinct `errored` bucket so a
`reason == "error"` infra failure (registry/manifest/cwd) is not mislabeled `assembly_failed`, and it breaks `clean`;
(c) the runnable script now truncates the sink at start (append-only reuse was overcounting persisted lines) and gates GO on
`clean AND matched == cohort_size` (not `clean` alone, which is true when nothing ran); (d) made `attempted` a derived
property; removed a dead variable + an unused import in tests. Verified and kept (deliberate): the soak loads registry/builder
per game (mirrors the real analyzer sidecar); the app-module cohort builders are not shared with tests (an app module cannot
import from tests). No correctness bug survived verification.

## 12. Remaining risks / gaps

- **Representative, not live.** The soak proves prompt byte-equivalence across all regimes on representative inputs; it does
  not exercise live retrieval, real data shapes, or volume. A live-data soak (when DB/network access is available) is the
  stronger evidence before any registry-authoritative routing.
- **Partial-evidence shapes** remain un-representable by the overlay templates (counted separately, fail-loud-no-capture). If
  these prove common in live data, an overlay slice must add partial-evidence variants before those regimes could route.
- **Enabled-mode cost** (registry/builder per request) unoptimized; cache deferred.
- No DB persistence (deliberate); JSONL sink single-writer. Still NOT live routing.

## 13. Next recommended slice

**Live-Data Shadow Soak v1 (when DB/network access exists):** run the soak against a real captured MLB slate (read-only, no
settlement/scoring), confirm zero mismatch/assembly/sink/error across the real regime distribution, and record the counts --
the live-data analogue of this representative soak. Then **Registry-Authoritative Prompt v1:** behind a separate default-off
flag, let the registry prompt become the model input ONLY for regimes with a proven-clean live soak, keeping the live builder
as the fallback + equality guard. Before that: add the module-level registry/builder cache and decide
`.NET sourceDepth -> dataRegime` regime ownership. Defer DB persistence, protocol-as-execution, buyer UI.
