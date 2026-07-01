---
title: "Prompt Provenance Calibration Export v1"
type: "export"
date: "2026-06-29"
status: "complete"
project: "DAI"
slice: "Prompt Provenance Calibration Export v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - provenance
  - calibration
  - metrics
related:
  - "06 Execution/exports/platform-side-prompt-provenance-calibration-export-v1.md"
  - "06 Execution/prompt-provenance-read-model-exposure-v1.md"
---

# Prompt Provenance Calibration Export v1

**status:** active doctrine (calibration export joins route provenance + optional artifact dump; JSON/CSV)
**date:** 2026-06-29

## purpose

Convert the live route provenance (exposed + persisted in Prompt Provenance Read-Model Exposure v1) into a
calibration-ready dataset so future cohorts can be grouped by prompt recipe, prompt version, data regime, prompt
source, and fallback status. The gap this closes: provenance was observable (header + JSONL) but not convenient
to analyse. This slice adds the smallest durable export surface; it is not a dashboard or analytics product.

## start state

dai `181ef59` (main, synced): Prompt Provenance Read-Model Exposure v1 pushed -- `app/prompting/route_provenance.py`
(`PromptRouteProvenance` + sinks + env-gated persist), `X-Prompt-Route-Provenance` header on /sports/analyze,
JSONL sink gated by `DAI_MLB_ROUTE_PROVENANCE_PATH`, `DEFAULT_ALLOWLIST` exactly four regimes. dai-vault
`6b372e3` (main, synced) with the four prior slice docs. Only the pre-existing untracked
`06 Execution/system-state-synopsis-v1.md` was dirty (unrelated, untouched).

## export source(s)

- **Primary:** the route-provenance JSONL written by the analyzer sink (`DAI_MLB_ROUTE_PROVENANCE_PATH`), read
  via `read_provenance_records` (= `JsonlRouteProvenanceSink.read_all`). This is sufficient for the route +
  run-anchor side of the export. JSONL stays append-only and is an audit/export aid, NOT the system of record.
- **Optional secondary:** an out-of-band artifact/outcome dump (json array or jsonl, keyed by `agentRunId`),
  read via `read_artifact_records`. Supplies decision/outcome fields. The analyzer/agent-service does not persist
  artifacts or outcomes (the .NET platform owns those), so this dump is operator-supplied until a .NET read
  model exists; absent it, decision/outcome fields export as null.

No DB, no network, no model call, no migration. The .NET layer persisting the header on the AgentRun row (and
exposing decision/outcome) is the deferred path that would later replace the operator-supplied artifact dump.

## export surface

- `app/services/route_calibration_export.py` -- pure shaping: `build_calibration_row(provenance, artifact)`,
  `join_calibration_records(provenance_records, artifact_records=None)` (outer join on `agentRunId`),
  `prompt_route_key(provenance)`, `rows_to_json`, `rows_to_csv`, `read_provenance_records`,
  `read_artifact_records`, and the canonical `CALIBRATION_FIELDS` order.
- `scripts/export_route_calibration.py` -- CLI wrapper (mirrors `export_mlb_soak_requests.py`).

Run from services/agent-service:

```
python scripts/export_route_calibration.py --provenance <route_prov.jsonl> --out <scratch>/calib.json
python scripts/export_route_calibration.py --provenance <route_prov.jsonl> \
    --artifacts <artifacts.json|.jsonl> --format csv --out <scratch>/calib.csv
```

Exit 0 iff at least one row was emitted. Write `--out` to scratch / out-of-repo (generated files are not
committed).

## output format

JSON array (default) or CSV (`--format csv`), both using the canonical `CALIBRATION_FIELDS` order. CSV: null ->
empty cell, booleans -> `true`/`false`.

## output schema

Identity / run: `agentRunId`, `createdUtc`, `competition`, `sport`, `externalGameId`, `homeTeam`, `awayTeam`,
`commenceTime`.
Prompt provenance: `promptSource`, `registryAuthoritativeEnabled`, `legacyFallbackUsed`, `regimeAllowlisted`,
`routingReason`, `fallbackReason`, `selectedDataRegime`, `selectedPromptRecipeId`, `selectedPromptVersion`,
`assembledHash`.
Derived: `promptRouteKey` = `selectedPromptRecipeId@selectedPromptVersion::selectedDataRegime` (or `unknown`).
Decision: `leanSide`, `confidence`, `advertisedStrength`, `evidenceSufficiency`, `sourceDepthBand`, `posture`.
Outcome / reconciliation: `outcomeStatus`, `actualWinner`, `resultSide`, `matchedOutcome`, `reconciledAtUtc`.

Decision/outcome come ONLY from the supplied artifact dump, via a bounded field allowlist (with snake/camel
aliases). Model reasoning / chain-of-thought / free prose (`summary`, `protocol`, `factors`, lean text) is never
copied -- route metadata + bounded decision/outcome only.

## grouping keys

The dataset is grouping-ready by: `selectedPromptRecipeId`, `selectedPromptVersion`, `selectedDataRegime`,
`promptSource`, `fallbackReason`, `legacyFallbackUsed`, and the combined `promptRouteKey`
(`recipe@version::regime`). A calibration pass groups outcomes by any of these (or joins to outcomes by
`agentRunId`).

## missing / historical provenance behavior

- A provenance record with no matching artifact -> decision/outcome fields null.
- An artifact run with no provenance (historical, before the sink existed) -> route fields null and
  `promptRouteKey="unknown"`; the run still exports (identity from the artifact).
- Missing files -> empty record sets (no crash); `read_provenance_records` on an absent JSONL returns `[]`.
- Nothing is fabricated: missing artifact, outcome, or route values are null/unknown, never guessed.

## outcome / reconciliation behavior

Outcome fields are copied only when already present in the supplied artifact dump. The agent-service has no
reconciliation/outcome store of its own, so without an artifact dump these are null. No outcome field is invented.

## tests run (venv python, from services/agent-service)

- `pytest tests/test_route_calibration_export.py -q` -> 10 passed.
- `pytest tests/test_route_calibration_export.py tests/test_route_provenance.py
  tests/test_route_provenance_exposure.py tests/test_prompt_route_decision.py tests/test_registry_prompt_canary.py
  tests/test_prompt_provenance.py tests/test_prompt_registry_contract.py tests/test_mlb_soak_export.py -q`
  -> 122 passed.
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- CLI smoke on a small fixture (2 provenance records + 1 artifact): JSON + CSV both emitted, exit 0,
  `unknown_route_rows=1` (the live/disabled run).
- `pytest -q` (full suite, final verification) -> 382 passed (was 372; +10 new), 0 failed.
- No paid model calls.

## buyer-facing impact

None. Export-only utility; no route/analyzer/endpoint change this slice; buyer artifact body/copy/UX untouched.

## paid-call status

None. No model call, network, or db access anywhere in the export path.

## risks / deferred items

- Decision/outcome data depends on an operator-supplied artifact dump until the .NET layer persists the route
  header on the AgentRun row and exposes decision/outcome through a read model (deferred:
  .NET AgentRun Prompt Provenance Persistence v1).
- JSONL provenance is single-writer and operator-gated (default-off); it is an audit/export aid, not durable
  platform storage.
- Historical runs predating the sink export with `promptRouteKey="unknown"`; no backfill.
- Export is MLB-centric (the only registry-routed competition producing route provenance today), though the
  shaping is competition-agnostic.

## rollback plan

Drop `app/services/route_calibration_export.py`, `scripts/export_route_calibration.py`, and
`tests/test_route_calibration_export.py`. Purely additive -- no existing code, schema, env, or contract changed,
so removal is clean.

## next recommended slice

.NET AgentRun Prompt Provenance Persistence v1 (store the `X-Prompt-Route-Provenance` header on the agentrun row
+ expose decision/outcome, removing the operator-supplied artifact dump), OR Broad Cohort Rerun Grouped by Prompt
Recipe v1, OR Live-Scheduled Starter-Missing Soak v1.

Related: [[prompt-provenance-read-model-exposure-v1]], [[default-allowlist-widening-v1]],
[[phase-3-2-global-prompt-routing-hardening-v1]], [[mlb-request-input-export-v1]].
