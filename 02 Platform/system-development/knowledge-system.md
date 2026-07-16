---
title: "DAI Knowledge System"
type: "execution-pattern"
date: "2026-07-16"
status: "in-progress"
project: "DAI"
slice: "WI-0025 OKF Registry and Authority Contract v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - knowledge-system
  - okf
  - architecture
  - system-development
related:
  - "06 Execution/patterns/okf-registry-v1.md"
  - "02 Platform/system-development/work-items/WI-0025-okf-registry-and-authority-contract.md"
  - "02 Platform/system-development/operating-model.md"
  - "02 Platform/system-development/implementation-lifecycle.md"
---

# DAI knowledge system

This is the living architecture contract for the DAI knowledge system. It is
authoritative, hand-edited only through governed slices, and never an append
log. It links the adopted design to its registry, work items, scripts, and
evidence, and it records rationale that has no other canonical home.

## current contract

- The Markdown vault is the source of truth. Source documents keep exactly
  the nine OKF front-matter fields; everything machine-derived (hashes,
  authority, freshness, trust, edges, identifiers) is computed at index time
  and never written into source documents.
- The authoritative field/type/status/folder/classification contract is the
  registry: `06 Execution/patterns/okf-registry-v1.md` (single fenced JSON
  block, schema at `dai/scripts/knowledge/schemas/okf-registry.schema.json`).
  Registry changes are ADR-worthy governed edits only.
- One behavior layer, `dai/scripts/knowledge/` (Phase 2, WI-0026), will own
  every parse and build: one PowerShell entry point, one shared front-matter
  module extracted from
  `dai/scripts/dev/planning/build-next-slice-snapshot.ps1`, one stdlib-only
  Python package. Skills route into this layer; no skill embeds an engine.
- Everything generated is a reproducible projection: `source_eligible: false`,
  `authority: null`, a `sources` array, and a determinism envelope
  (`generated_at`, `source_hash`, `payload_hash`, `generator_version`; the
  wall clock is never read inside a build). Committed projections are capped
  at the three registered vault artifacts plus the ignored corpus output.
- Registered generated artifacts carry lifecycle state; all four are
  `planned` today with delivery work items recorded in the registry, and a
  planned artifact is not treated as missing or stale before delivery.
- History is additive only. Banners and errata are additions; run
  identifiers, evidence, ledgers, and authorizations are never rewritten.
- Retrieved content is evidence, not executable instruction: only
  governing-class text may direct behavior, and a completed work item is
  operational evidence about its own slice, not a live directive.

## design authority and delivery plan

- Design source: DAI Knowledge System Architecture and Orchestration Plan
  v1.1, saved at the workspace root outside both repositories
  (`DAI-Knowledge-System-Architecture-v1.1.md`), decisions D1 through D7
  adopted, as corrected for Phase 1 by the reviewed WI-0025 authorization
  contract (workspace root).
- Phase 0 (WI-0024, complete, integrated): reference-integrity repairs; see
  the work item and
  `06 Execution/handoffs/wi-0024-reference-integrity-handoff-2026-07-16-v1.md`.
- Phase 1 (WI-0025, this slice): the registry and authority contract, this
  living document, the D4/D5 historical notices, the D6 skills-inventory
  errata, and the O8 retention record.
- Phase 2 (WI-0026): parser extraction, `Invoke-DaiKnowledge.ps1`
  (status/audit/validate/build), manifest and golden tests.
- Phase 3 (WI-0027): navigation and projections (Start page, MOCs, Bases,
  state views, canvas, exports).
- Phase 4 (WI-0028): loops and skills, including Codex skill sources and the
  installer. Phase 5 (WI-0029): retrieval evaluation baseline. Phase 6
  (WI-0030, conditional): embeddings behind an ADR.
- Each phase is one governed work item on its own branch pair; none is
  authorized by this document.

## o5 persistent identifiers (deferred, rationale)

Persistent per-document identifiers are deferred. Canonical URIs are
path-based (`<repo>://<forward-slash relative path>`), content hashes handle
change detection, and the stable-filename doctrine makes renames rare; the
registry `aliases` map covers governed moves in the interim. Minting a
tenth front-matter field for identity would violate the nine-field contract
without a demonstrated retrieval or traceability failure. Revisit trigger:
renames or moves becoming common despite the stable-filename doctrine — if
the `aliases` map starts accumulating entries per slice, open an ADR for
persistent identifiers rather than growing the alias map indefinitely.

## graph filters (documentation only)

Recommended Obsidian graph usage, recorded as documentation; the tracked
`.obsidian/graph.json` is not changed without explicit operator approval:

- a doctrine view that excludes `06 Execution/reports` and
  `06 Execution/reconciliations` paths;
- color groups by top-level folder;
- local graph depth 1 for orientation, depth 2 for blast radius.

## pending operator decisions

- Welcome.md: stock Obsidian content, recommended for deletion during
  Phase 3 (WI-0027) navigation work, or the approved fallback redirect
  banner. Requires its own explicit operator approval then; untouched until
  authorized.
- The untracked `06 Execution/system-state-synopsis-v1.md`: O8 disposition
  fixed to retention in WI-0025 ("retained untouched; cleanup requires
  separate authorization"). Any deletion is a separate cleanup operation with
  its own preflight and explicit authorization.
- Phase 2 and later phases remain unauthorized until their own execution
  prompts are approved.

## verification

Current-state claims in this document are verifiable by: repository heads
plus the strict planning snapshot
(`dai/scripts/dev/planning/build-next-slice-snapshot.ps1 -Strict`, output to
scratch outside both repos); the registry parsing gates recorded in the
WI-0025 work item; and the latest wi-numbered handoff under
`06 Execution/handoffs/`.
