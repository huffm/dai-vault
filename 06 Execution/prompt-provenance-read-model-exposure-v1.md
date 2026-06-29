# Prompt Provenance Read-Model Exposure v1

**status:** active doctrine (live-path route provenance projected, exposed via response header, env-gated persist)
**date:** 2026-06-29

## purpose

Persist and expose prompt route provenance so future calibration can group artifacts by prompt recipe, prompt
version, data regime, and fallback status. Phase 3.2 produced a typed `PromptRouteDecision`, but it was only
logged in-process. This slice projects that decision into a stable, serializable read model, exposes it on the
analyze response, and provides an env-gated durable sink -- without changing the buyer artifact, the model-call
count, or `DEFAULT_ALLOWLIST`.

## start state

dai `5395e5e` (main, synced): Default Allowlist Widening v1 committed (4-regime allowlist), Phase 3.2
`PromptRouteDecision` / `decide_model_prompt` / `select_model_prompt` shim present, analyzer logs the route
decision. dai-vault `e085705` (main, synced) with the three prior slice docs. Only the pre-existing untracked
`06 Execution/system-state-synopsis-v1.md` was dirty (unrelated, untouched).

## route metadata fields (the read model)

`app/prompting/route_provenance.py :: PromptRouteProvenance` (frozen, `extra="forbid"`) -- a projection of
`PromptRouteDecision` plus the run anchor:

- `promptSource` (live | registry)
- `registryAuthoritativeEnabled`
- `legacyFallbackUsed`
- `regimeAllowlisted`
- `routingReason`
- `fallbackReason` (disabled | regime_not_allowlisted | mismatch | assembly_error | error | null)
- `selectedDataRegime`
- `selectedPromptRecipeId`
- `selectedPromptVersion`
- `assembledHash`
- `agentRunId` (run anchor -- ties to the agentrun row the .NET layer persists)
- `competition`
- `createdUtc` (injectable; defaults to `utc_now_iso()`)

This is the LIVE analogue of the shadow-only `PromptProvenance`. The shadow record audits a shadow recipe
assembly and fails closed for live mode; this projection records the route decision that actually governed a
live model call (registry-authoritative OR live/legacy fallback), so it is NOT mode-gated. It is metadata about
prompt selection only -- never model reasoning, buyer advice, confidence, source depth, or a calibration
outcome.

## persistence location

agent-service owns no database (the .NET platform persists agent runs), so persistence follows the established
no-DB pattern: an append-only JSONL sink (`JsonlRouteProvenanceSink`, with `read_all()` for read-back) plus an
in-memory sink for tests, behind a `RouteProvenanceSink` protocol. Durable local capture is **env-gated and
default-off**: `persist_route_provenance(provenance)` appends to the file at `DAI_MLB_ROUTE_PROVENANCE_PATH` when
set, and is a pure no-op (no file i/o) when unset, so the live request path is unchanged unless an operator opts
in. Unlike the shadow `JsonlProvenanceSink`, this sink is not mode-gated (a live route decision is exactly what
it persists).

## read-model / API exposure

The canonical cross-layer exposure is the response header `X-Prompt-Route-Provenance` (JSON projection) set by
`POST /sports/analyze` for MLB runs, anchored by the `X-Agent-Run-Id` request header. The .NET persistence layer
can read this header and store it on the agentrun read model with no change to the buyer artifact body. Wiring:
`analyze_mlb` now computes the route decision for every run and hands it to an optional `on_route_decision`
callback; the route handler's callback builds the `PromptRouteProvenance`, sets the header, and calls the
env-gated persist. The buyer artifact (`SportsAnalysisResponse`) is untouched; non-MLB competitions
(not registry-routed) carry no header.

## historical / null behavior

Agent runs created before this slice have no route provenance: there is no header on historical responses and no
JSONL line. `JsonlRouteProvenanceSink.read_all()` returns `[]` for a missing file, and a calibration reader must
treat absent provenance as "unknown route" rather than an error. New runs always carry the header; the JSONL
line exists only when the operator has configured the path env.

## migration details

None. No schema migration, no DB column, no SQL change. Persistence is the agent-service JSONL pattern; durable
storage of the cross-layer header on the agentrun row is the .NET layer's existing responsibility and is out of
scope here (documented as the consumer of the header).

## test coverage

New suites (16 tests):

- `tests/test_route_provenance.py` (9): projection serializes to a stable, exact JSON shape; projections from a
  registry-authoritative decision, a default-off (disabled) decision, and a non-allowlisted fallback decision;
  the projection is frozen + closed; in-memory and JSONL sinks append in order; JSONL round-trips via
  `read_all`; `read_all` on a missing file returns `[]`; env-gated persist is a no-op by default and persists
  when the path env is set.
- `tests/test_route_provenance_exposure.py` (7): `analyze_mlb` invokes the callback for every run (default-off:
  `live`/`disabled`) with a single model call; registry-authoritative decision reaches the callback; a callback
  failure is non-fatal to the request; the route sets `X-Prompt-Route-Provenance` for MLB with the buyer body
  unchanged; the header is registry-authoritative when the canary is enabled (default-allowlisted regime);
  env-path persistence writes a readable line; non-MLB runs carry no header.

Commands run (venv python, from services/agent-service):

- `pytest tests/test_route_provenance.py tests/test_route_provenance_exposure.py -q` -> 16 passed.
- `pytest tests/test_prompt_route_decision.py tests/test_registry_prompt_canary.py tests/test_prompt_provenance.py
  tests/test_prompt_registry_contract.py tests/test_route_provenance.py tests/test_route_provenance_exposure.py
  tests/test_sports_analyzer.py -q` -> 207 passed.
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- `pytest -q` (full suite, final verification) -> 372 passed (was 356; +16 new), 0 failed.
- No paid model calls.

## buyer-facing impact

None. The buyer artifact body (`SportsAnalysisResponse`) is unchanged; provenance lives on a response header and
an optional internal JSONL file. No UX/copy/pricing/billing/auth/tenant/dashboard/confidence/model change.

## calibration usage

A later calibration pass joins the persisted route provenance (or the .NET-stored header) to outcomes by
`agentRunId` and groups by `selectedPromptRecipeId` / `selectedPromptVersion` / `selectedDataRegime`, and filters
by `promptSource` / `legacyFallbackUsed` / `fallbackReason` to separate registry-authoritative runs from
live/legacy fallbacks. This answers: which recipe/version/regime produced an artifact, whether fallback was used,
and whether the run was registry-authoritative.

## risks / deferred items

- The header is the cross-layer exposure; durable storage of it on the agentrun row is the .NET layer's job and
  is not implemented in this slice (the python side exposes + optionally local-persists only).
- Local JSONL persistence is single-writer and default-off; it is an audit/calibration aid, not the system of
  record.
- Historical runs have null provenance (documented); no backfill.
- Route provenance is produced for MLB only (the sole registry-routed competition today).

## rollback plan

Smallest revert: remove the `on_route_decision` argument + callback in `app/routes/sports.py` (and the `Response`
param/header constant), revert the `analyze_mlb` block to call the canary only when enabled, and drop
`app/prompting/route_provenance.py` + its two test files and the `__init__.py` exports. No data migration, no env
change. Operators can also simply leave `DAI_MLB_ROUTE_PROVENANCE_PATH` unset to disable durable persistence
while keeping the header.

## next recommended slice

Live-Scheduled Starter-Missing Soak v1 (confirm the promoted regimes on genuine live games when the stack is up),
OR Prompt Provenance Calibration Export v1 (join route provenance to outcomes and export grouped metrics), OR a
Broad Cohort Rerun Grouped by Prompt Recipe v1.

Related: [[phase-3-2-global-prompt-routing-hardening-v1]], [[default-allowlist-widening-v1]],
[[starter-missing-registry-canary-confirmation-v1]], [[prompt-provenance-persistence-v1]].
