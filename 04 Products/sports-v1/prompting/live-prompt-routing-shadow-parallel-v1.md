# Live Prompt Routing (Shadow-Parallel) v1

**date:** 2026-06-28
**status:** IMPLEMENTED (DEFAULT-OFF) + VERIFIED. Adds a default-off shadow-parallel prompt validation sidecar INSIDE the
real MLB analyzer request flow: when enabled it builds the live prompt as today, builds the shadow registry prompt, asserts
byte equality, and captures provenance after equality -- while the live prompt remains the only prompt sent to the model. No
second model call, no second artifact, no buyer-facing change. When disabled (the default) the request path is byte-identical
to before. Also extracts the shared MLB prompt formatters so the live path and the migration oracle no longer duplicate phrase
construction. `.NET` unchanged.
**type:** Live Prompt Routing (Shadow-Parallel) v1 slice. test-driven-development + verification-before-completion +
code-review (high). The dai-specific slice skills referenced in `dai/CLAUDE.md` (dai-skill-router / dai-slice-runner /
dai-grill-with-vault / dai-docs-architect / dai-agent-handoff) are NOT installed in this environment; superpowers equivalents
+ a manual structured handoff were substituted (recorded in the Skills Gate note of the handoff).
**anchor:** put the shadow registry prompt next to the authoritative live prompt ON REAL REQUESTS, behind an explicit
default-off gate, prove byte equality and capture provenance -- without the validation ever altering, delaying-incorrectly, or
breaking the live analysis. This is the controlled step before live prompt routing could ever flip; it is still NOT live
routing.

See also: [[live-migration-readiness-v1]], [[prompt-provenance-persistence-v1]] (Phase 4.1),
[[prompt-provenance-and-manifest-integrity-v1]] (Phase 4), [[market-depth-starter-named-overlays-v1]] (Phase 3.2),
[[data-regime-overlay-mlb-prompt-equivalence-v1]] (Phase 3.1), [[prompt-assembly-engine-v1]] (Phase 3),
[[prompt-registry-contract-v1]] (Phase 2), [[controlled-dynamic-prompt-assembly-architecture-v1]] (Phase 1).

## 1. What changed in sports_analyzer.py

Two changes, both behavior-preserving for the live prompt bytes (proven by the unchanged 9-regime equivalence suite):

1. **Extracted shared MLB formatters** (the one source of truth for the numeric/threshold rules, so the live path and the
   migration oracle can never silently drift):
   - `format_pitcher_quality` -- the prior `_format_pitcher_quality`, promoted to a public name. In-module/live callers
     (`build_mlb_user_message`) now use the public name; `_format_pitcher_quality` is kept as a thin alias for existing test
     imports only (same object, byte-identical output).
   - `format_pct_whole(fraction)` -> `f"{fraction*100:.0f}"`; `format_one_decimal(value)` -> `f"{value:.1f}"`;
     `mlb_disagreement_strength(range)` -> `"books largely agree"` if `range < 0.05` else `"books disagree noticeably"`.
   - `build_mlb_user_message`'s market-depth block now calls these instead of inline literals. Byte-identical output (tests 7,
     8 + the depth-substring test assert this).
2. **Wired the default-off shadow sidecar** into `analyze_mlb`: it gains a keyword-only `shadow_config:
   ShadowValidationConfig | None = None`. After building the live `user_msg` (unchanged), if a config resolves to enabled it
   runs `run_mlb_shadow_validation(..., live_prompt=user_msg)` as a side-effect; the live `user_msg` is always what goes to
   `_call_model`. `shadow_validation` is imported LAZILY inside `analyze_mlb` (to avoid a module-load cycle
   `shadow_validation -> migration_readiness -> sports_analyzer`, and so an explicitly-disabled injected config imports
   nothing). The `"ShadowValidationConfig | None"` annotation is a string (the file has `from __future__ import annotations`),
   so it does not import at runtime.

## 2. Why the live prompt remains authoritative

`build_mlb_user_message` is called exactly once and its output (`user_msg`) is the only prompt passed to `_call_model`. The
shadow registry prompt is built separately for comparison only; it never replaces `user_msg`. There is no second model call
and no second artifact. The shadow path stays shadow-only (the registry recipes + `assemble_recipe_for_migration` fail closed
for `mode="live"`).

## 3. Default-off shadow flag behavior

`app/services/shadow_validation.py` (NEW):
- `ShadowValidationConfig(enabled: bool = False, sink_path: str | None = None)` -- default DISABLED.
- `load_shadow_validation_config(env=None)` -- reads `DAI_MLB_SHADOW_VALIDATION` (truthy: `1/true/yes/on`, case-insensitive;
  anything else incl. unset = disabled) and optional `DAI_MLB_SHADOW_SINK_PATH`. Injectable for tests.
- `run_mlb_shadow_validation(config, ...)` -- the sidecar. Returns immediately (`ShadowValidationOutcome(ran=False)`) when
  disabled, constructing NOTHING (no registry, no builder, no sink). When enabled it builds the shadow prompt, asserts byte
  equality, and captures provenance to the JSONL sink (only if `sink_path` is set -- the sink is optional) AFTER equality.

**Disabled mode is byte-identical (proven):** with an explicit disabled config, `analyze_mlb` imports neither
`load_shadow_validation_config` nor `run_mlb_shadow_validation` and the `user_msg`/`_call_model` arguments are unchanged
(test: `test_disabled_mode_is_byte_identical_and_runs_no_shadow`). With no config injected and the env flag unset, the env
default resolves disabled and the sidecar does not run (`test_default_env_path_is_disabled`). Disabled mode never constructs
or writes a sink.

## 4. When provenance is emitted

Only when ALL of: the flag is enabled, a `sink_path` is configured, AND byte equality was proven first. Capture uses the
provenance built from the single shadow compose (downstream, never re-rendered). On any readiness gap nothing is captured.

## 5. Failure surfacing (enabled mode) -- loud, but never breaks the live request

The entire sidecar body (registry/builder/sink construction + the equality check) runs under one guard so that NOTHING --
a byte mismatch, an un-assemblable partial-evidence input, a manifest load error, a wrong working directory, or a sink write
failure after equality -- can raise into the live request. Each failure is logged at error level (observable, NOT silently
swallowed) and returned in `ShadowValidationOutcome.error`; the authoritative live analysis is unaffected. Distinct log
markers: `shadow prompt validation MISMATCH` (byte divergence, with regime), `could not assemble` (partial-evidence input),
`FAILED (non-fatal to live request)` (any other error). The standalone oracle `check_mlb_shadow_equivalence` still RAISES
(fail loud) for harness/test use; the analyzer sidecar is the production-style wrapper that logs-and-continues. Tests assert
all three loud paths plus that `analyze_mlb` completes with the authoritative prompt even when the sidecar errors.

## 6. Why DB persistence remains deferred

Unchanged from [[prompt-provenance-persistence-v1]] sec 2/6: the agent service has no DB layer (the .NET platform owns
persistence); a DB sink now would force a premature schema + cross-stack runtime coupling. The sidecar writes through the
existing `ProvenanceSink` abstraction (append-only JSONL only, for now), so a future DB sink drops in unchanged. The documented
future contract (append-only `prompt_provenance` table, non-null `tenantId`, audit key `(tenantId, createdUtc, assembledHash)`,
never updated/deleted, tenant-scoped reads) still stands.

## 7. Why .NET remains unchanged

No .NET change was required. Regime ownership stays with PYTHON (`dataregime.py`); the shadow route/slots are derived in
python from the analyzer inputs. The designed `.NET sourceDepth -> dataRegime` mapping
([[prompt-provenance-persistence-v1]] sec 8) remains deferred to a future slice that would decide regime ownership before any
production live routing.

## 8. Files changed (all in `dai`)

- `app/services/sports_analyzer.py` -- promoted `format_pitcher_quality` (+ alias); added `format_pct_whole`,
  `format_one_decimal`, `mlb_disagreement_strength`; `build_mlb_user_message` depth block uses them; `analyze_mlb` gained the
  default-off `shadow_config` param + lazy sidecar call.
- `app/services/shadow_validation.py` (NEW) -- config + sidecar.
- `app/services/migration_readiness.py` -- uses the shared helpers; `check_mlb_shadow_equivalence` gained optional
  `live_prompt` (so the analyzer's already-built prompt is reused -- single live build).
- `tests/test_shadow_validation.py` (NEW, 16 cases).
- No change to `provenance.py`, `provenance_sink.py`, `integrity.py`, `dataregime.py`, `builder.py`, `manifest.json`,
  templates, or `.NET`.

## 9. Exact tests run + results (venv python, from `services/agent-service`)

- `pytest tests/test_shadow_validation.py -q` -> **16 passed**.
- `pytest tests/test_mlb_prompt_equivalence.py tests/test_mlb_branch_overlays.py -q` (9-regime equivalence, the
  byte-preservation guard for the extraction) -> **30 passed** (unchanged).
- `pytest tests/test_migration_readiness.py -q` -> **17 passed** (unchanged).
- `pytest -q` (full agent-service suite) -> **278 passed, 0 failed** (262 prior + 16 new).
- `python scripts/check_prompt_manifest.py` -> `OK ... (8 templates, 9 recipes)`, exit 0.
- `git status` confirms `sports_analyzer.py` + `migration_readiness.py` modified plus two new files; `manifest.json` /
  templates / `.NET` unchanged.

## 10. Code review

`/code-review high` (8 finder angles + verify). Resolved: (a) wrapped the ENTIRE sidecar (construction + check) in a broad
guard so an enabled sidecar can never break the live request -- a manifest/cwd error or a post-equality sink-write OSError is
now logged loudly + returned, not raised (added two tests + an end-to-end analyzer test proving non-fatal); (b) migrated the
in-module `build_mlb_user_message` callers to the public `format_pitcher_quality` so the alias is load-bearing only for test
imports (with an accurate comment). Verified and kept (deliberate): the env-var fallback alongside the injected config (the
established service config pattern; the injected param is the DI path); `ShadowValidationOutcome.equal` (NOT redundant here --
a mismatch is a returned state, not an exception). No correctness bug survived verification.

## 11. Remaining risks / gaps

- **Enabled-mode cost.** When enabled, `run_mlb_shadow_validation` loads the registry + builds the builder (file/AST reads)
  per request, synchronously around the model call. Acceptable for a default-off canary/soak mode; a module-level lazy cache
  is the obvious optimization if it is ever run hot. Documented, not implemented.
- **cwd-relative templates dir.** The builder uses the repo-relative `app/prompting/templates` (the codebase norm). With the
  broad guard, a wrong cwd is a logged non-fatal outcome, not a request break -- but enabled mode still needs the right cwd
  to actually validate.
- **Env read inside the service function** (`load_shadow_validation_config` when no config injected). Consistent with the
  service's existing env-config pattern (e.g. `OPENAI_API_KEY`); a future composition root could resolve the config once and
  pass it in.
- **Partial-evidence input shapes** still cannot be represented by the overlay templates (fail-loud, no capture). MLB only.
- No DB persistence (deliberate). JSONL sink single-writer. Still NOT live routing -- the shadow prompt never reaches the
  model.

## 12. Next recommended slice

**Shadow-Parallel Cohort Soak v1 (operational, no code-as-routing):** with the flag enabled in a controlled (dev/canary)
environment and a configured JSONL sink, run a real MLB cohort through the analyzer, confirm zero `MISMATCH` / `FAILED` logs
and a clean provenance file across regimes, and record the result -- the empirical evidence required before live routing could
be considered. Then (code) **Registry-Authoritative Prompt v1:** behind a separate default-off flag, let the registry prompt
become the model input ONLY for regimes with a proven-clean soak, keeping the live builder as the fallback + equality guard.
Before that: add the module-level registry/builder cache, and decide `.NET sourceDepth -> dataRegime` regime ownership. Defer
DB persistence, protocol-as-execution, buyer UI.
