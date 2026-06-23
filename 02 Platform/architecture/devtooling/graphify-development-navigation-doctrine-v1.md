# graphify development navigation doctrine v1

**date:** 2026-06-23
**status:** active development-tooling doctrine — local navigation layer only, not runtime, not product
**scope:** applies to agent-assisted development on `<DAI_REPO_ROOT>`; does not apply to `<DAI_VAULT_ROOT>`

---

## purpose

Graphify is a local code-navigation tool. It builds an AST-derived graph of the
code repo (`graph.json`, `graph.html`, `GRAPH_REPORT.md`) so an agent can orient
in the codebase before editing, instead of discovering structure by repeated
grep, find, and one-file-at-a-time reads.

It supports the factory model by improving development throughput and agent
context retrieval. It is a faster map of the factory floor. It does not add a
product feature, it does not change what the factory builds, and it does not
expand what agents are allowed to do.

---

## positioning

- navigation layer — a map of code topology, queried during development.
- not a source of truth — it can be stale or name a relationship that does not hold.
- not a runtime dependency — nothing in the platform reads the graph at run time.
- not a buyer-facing product capability — it never appears in a niche output.
- not an agent permission expansion — it grants no new tools, data, or write access.

---

## strategic fit

- platform = factory.
- agents = workers.
- graphify = the factory floor map / development navigation aid.

It helps a worker understand code topology before changing a station, and it
makes blast radius visible before a DTO, artifact, handler, read model, or
contract change. It reduces repo spelunking and lowers the chance of an edit that
silently breaks a cross-stack contract. It does not participate in cognition, in
the artifact, or in any niche assembly line.

---

## approved uses

- orientation before an unfamiliar slice.
- blast-radius inspection before changing a DTO, artifact, handler, read model, or contract.
- cross-stack contract discovery across C#/.NET, Python FastAPI, and Angular.
- locating high-degree symbols and candidate architecture hotspots.
- comparing graph findings against source, tests, and vault docs.

---

## disallowed uses

- treating graph output as authoritative.
- making behavior claims from graph output alone.
- replacing tests, source inspection, or vault contracts with the graph.
- auto-generating code changes from graph findings without verification.
- enabling semantic / cloud / LLM extraction without explicit approval.
- running graphify against `<DAI_VAULT_ROOT>` until a separate docs-graphing strategy is approved.
- committing generated graph artifacts (`graphify-out/` is git-ignored).

---

## slice workflow

- start of an unfamiliar slice: run or inspect graphify for orientation.
- during planning: use `query` / `explain` / `path` / `affected` to scope the work.
- before edits: verify the actual source files, not the graph.
- before completion: verify tests and observed runtime behavior.
- after non-trivial code changes: `graphify update .` (AST-only, no API cost).
- after large refactors or deletions: `graphify update . --force`.

---

## truth hierarchy

authority runs top to bottom. lower tiers never override higher ones.

1. runtime behavior and tests.
2. source code.
3. explicit contracts and vault docs.
4. slice handoffs.
5. graphify output — navigation evidence only.

---

## current constraints

- AST-only local graph.
- no LLM extraction.
- no API cost.
- no code upload; extraction is fully local.
- communities are unnamed (no LLM to label them).
- cross-language edges are useful but may be name-based false positives.
- god-node rankings may be inflated by test fixtures.

---

## deferred decisions

each is explicitly out of scope until separately approved.

- LLM semantic extraction.
- running graphify over `<DAI_VAULT_ROOT>`.
- a combined workspace-level graph (code + vault).
- git hook automation.
- CI graph generation.
- making graph output available to other agents.

---

## initial graph-assisted investigations

these are the first three investigations worth running. each is navigation, not a
finding. nothing here is a behavior claim until verified against source and tests.

### 1. cross-stack sports contract coupling

- why it matters: the graph surfaces C# `DevCore.Api` handlers referencing the
  Python `services/agent-service/app/models/sports.py` and the Angular
  `agent-run.model.ts`. these are the shared-contract seams most likely to break
  silently across stacks.
- useful commands:
  - `graphify affected "SportsAnalysisResponse"`
  - `graphify path "AnalysisSportsMatchupReadHandler" "SportsAnalysisResponse"`
  - `graphify explain "SharpPublicContext"`
- required verification: read the real model definitions in `sports.py`, the
  matching C# DTOs/handlers under `platform/dotnet/DevCore.Api/Tools/Handlers`,
  the Angular models, and the contract tests (e.g. `ToolGatewayAnalyzeTests`).
  cross-check against `current-agent-run-contract.md`.
- expected outcome: a verified map of where the sports contract is shared across
  the three stacks, and confirmation that each edge is a real contract dependency,
  not a name collision.

### 2. AppDbContext schema/change blast radius

- why it matters: `AppDbContext` is a high-degree DB seam. any schema or migration
  change ripples through stores and read models, and a missed dependency can break
  persistence.
- useful commands:
  - `graphify affected "AppDbContext"`
  - `graphify explain "AppDbContext"`
- required verification: read `platform/dotnet/DevCore.Data/AppDbContext.cs`, the
  store classes (e.g. `ProbeRefreshMergeAuditStore`), and EF migration usage.
  confirm runtime against the dev SQL container before any migration.
- expected outcome: a verified list of everything that touches the EF context, used
  as a pre-change checklist before a schema or migration slice.

### 3. high-degree python helpers `_parse_response()` and `_make_raw()`

- why it matters: these top the god-node ranking. that may reflect real
  responsibility concentration in the agent service, or it may be test-fixture
  inflation. either answer is useful.
- useful commands:
  - `graphify explain "_parse_response"`
  - `graphify explain "_make_raw"`
- required verification: read the actual functions in `services/agent-service` and
  their tests. determine whether centrality reflects production reuse or test
  scaffolding.
- expected outcome: a decision on whether either helper is a refactor candidate, or
  simply a widely-reused, well-scoped helper that the ranking over-weights.

---

## related

- factory model and worker framing: `../cognitive-worker-doctrine.md`
- agent-run contract: `../current-agent-run-contract.md`
- committed project-level navigation guidance lives in `<DAI_REPO_ROOT>/CLAUDE.md`.
