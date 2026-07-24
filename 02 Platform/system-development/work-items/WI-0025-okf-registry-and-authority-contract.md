---
title: "WI-0025 OKF Registry and Authority Contract v1"
type: "plan"
date: "2026-07-16"
status: "complete"
project: "DAI"
slice: "WI-0025 OKF Registry and Authority Contract v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - okf
  - registry
  - knowledge-system
related:
  - "06 Execution/patterns/okf-registry-v1.md"
  - "02 Platform/system-development/knowledge-system.md"
  - "02 Platform/system-development/work-items/WI-0024-reference-integrity-repairs.md"
---

# WI-0025 OKF registry and authority contract v1

## problem  <!-- LITE -->

The vault has an OKF convention (nine front-matter fields, type-folders,
enum statuses) but no machine-readable contract: no registry document, no
schema, no recorded authority/freshness/trust semantics, and no living
architecture doc linking the knowledge-system design to its artifacts.
Three stale current-state documents (`dai/docs/current-state.md`,
`dai/docs/session-handoff.md`, `06 Execution/prompting/next-slice.md`) still
present themselves as current; `current-slice.md` carries no notice that it
is an append-only historical registry; and the skills-inventory update log
cites two pre-migration paths repaired elsewhere by WI-0024.

## desired behavior

One governed registry document with a single fenced JSON block, parseable by
PowerShell and Python and validated by a checked-in JSON Schema, records the
OKF contract including authority precedence, freshness, trust, projection
rules, exemptions, canonical type-folders with a legacy date gate, and
generated-artifact lifecycle. A living `knowledge-system.md` contract links
design to artifacts and records the O5 rationale. Historical and rolling
documents carry additive notices so no stale document claims current truth.

## affected surfaces

dai-vault created: `06 Execution/patterns/okf-registry-v1.md`,
`02 Platform/system-development/knowledge-system.md`, this work item,
`06 Execution/handoffs/wi-0025-okf-registry-handoff-2026-07-16-v1.md`.
dai-vault additive edits: `06 Execution/handoffs/current-slice.md` (D4 banner
+ closeout append), `06 Execution/prompting/next-slice.md` (D5 banner),
`06 Execution/skills/dai-skills-inventory-v1.md` (D6 errata).
dai created: `scripts/knowledge/schemas/okf-registry.schema.json`.
dai additive edits: `docs/current-state.md`, `docs/session-handoff.md`
(D5 banners).

## authority and design sources

- design source: DAI Knowledge System Architecture and Orchestration Plan
  v1.1, Phase 1 (workspace root, outside both repos), subject to the
  controlling review corrections in the reviewed authorization contract
- execution scope: `WI-0025-OKF-Registry-and-Authority-Contract-Authorization-Prompt.md`
  (workspace root; sha256 51B04FEA7EE288004B90290D51B2B3E1F8F5BF823232DC228F9EFA43965102FC,
  32338 bytes, 629 lines); the narrower contract controls on any conflict
- operator authorization: WI-0025 Phase 1 execution authorization, 2026-07-16

## exact scope

10 files, two repositories, docs only: 5 created (registry doc, living
contract, schema, this work item, the handoff) and 5 additively edited
(D4 banner + closeout in current-slice.md; D5 banners in next-slice.md,
current-state.md, session-handoff.md; D6 errata in the skills inventory).
Controlling review corrections applied: 9-field OKF front matter for new
platform doctrine; `type_folders` as arrays of canonical parents with
`exact-parent-directory` matching and `legacy_type_folder_ok_before`
date gate; generated artifacts carry `state` + `available_from_work_item`
(all four `planned`); D4/D5 banners reference only existing paths and live
commands; O8 fixed to retention.

## non-goals

No push, merge, pull request, or remote mutation. No deletion, move, rename,
or archive (Welcome.md and the untracked synopsis remain untouched). No
parser extraction, validator, Python package, views, Bases, Canvas,
retrieval, embeddings, or Codex skills (Phase 2+, WI-0026..WI-0030). No edit
to any existing document's front matter. No mass migration or re-typing. No
runtime, prompt, routing, schema-of-record, product, or test behavior
change. No `06 Execution/state/` folder creation. No persistent assistant
memory writes.

## protected pre-existing state

- dai `platform/dotnet/DevCore.Data/DevCore.Data.csproj`: modified (stat
  anomaly, empty diff), sha256 63EF248822D14E458C83C260BB6EC9E7CAEA842F345DD451F85F1B27C1D16A8F;
  never edited, staged, or reconciled.
- dai-vault `.obsidian/graph.json`: sha256
  2313F26F836E156EB2813506C67D7AC499E76AC7A12EC8456F011EDCB643884A at
  preflight (obsidian closed, hash stable across immediate re-read); never
  staged.
- dai-vault untracked: `06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json`
  (68948EBD...) and `06 Execution/system-state-synopsis-v1.md` (25835E6C...);
  never modified, staged, or deleted. O8 disposition: retained untouched;
  cleanup requires separate authorization.

## acceptance criteria  <!-- LITE -->

- Registry JSON parses in PowerShell `ConvertFrom-Json` and Python
  `json.loads` with exactly seventeen top-level keys; exactly one fenced
  json block exists in the registry document.
- `Test-Json` validates the block against the checked-in draft 2020-12
  schema; the schema file itself parses as JSON in both parsers.
- Strict planning snapshot: 19 work items (WI-0025 added), 0 continuations,
  0 warnings; deferred/timeline deltas explainable by this slice alone.
- Additive integrity: current-slice.md final file minus the two insertions
  reproduces the preflight baseline byte-for-byte (sha256
  5F2A3FEE95A28301D9A173574CEF1D76BC3CA70E8C4919C24321F0102601252B,
  1400653 bytes); the five bannered/errata files show insertion-only diffs.
- D4/D5 banners reference no not-yet-created path (no current-state-view).
- Every `related:` entry in the new documents resolves; the four new OKF
  documents parse in the snapshot without warnings.
- `git diff --check` clean; staged sets equal the allowlists exactly
  (7 vault paths, 3 dai paths); protected baselines unchanged; commits exist
  only on `wi/0025-okf-registry`; nothing pushed or merged.

## test plan  <!-- LITE -->

Docs-only slice: no code tests. Verification is the command battery below
plus the OKF type/folder inventory recorded at preflight (19
execution-pattern documents = 10 platform + 9 patterns; 24 noncanonical
active-type documents, all dated before 2026-07-16, none on/after).

## verification commands  <!-- LITE -->

- `pwsh dai/scripts/dev/planning/build-next-slice-snapshot.ps1 -OutputPath <scratch>/wi-0025-snapshot.json -Strict`
- PowerShell and Python registry parsing gates (regex-extract the fenced
  block; seventeen keys; schema_version 1.0)
- `Test-Json -Json <block> -SchemaFile dai/scripts/knowledge/schemas/okf-registry.schema.json`
- byte-reconstruction proof for current-slice.md; insertion-only diff review
  for the five edited files
- `git diff --check` and staged-set comparison against the allowlists
- sha256 re-hash of the four protected files against preflight baselines
- Test-Path resolution of every `related:` entry in the new documents

## risks

Low. Docs-only; the snapshot parser reads the new front matter (mitigated by
mirroring the exact quoting convention of neighboring canonical documents
and gating on the strict snapshot); byte-splice insertions into five
historical files (mitigated by the reconstruction proof and insertion-only
diff review); Test-Json draft 2020-12 support (verified on pwsh 7.6.1).

## rollback

All changes are local commits on `wi/0025-okf-registry` in both repos;
rollback is deleting the branches before integration (or `git revert` after).
Banners are additive, so reverting restores exact prior bytes. The five
created files are new; rollback removes them cleanly. No generated
artifacts, no remote effects, no pushes.

## links  <!-- LITE; all 8 required at close, per work-item-traceability -->

- work item: WI-0025
- branch: wi/0025-okf-registry (dai and dai-vault)
- pr: none (push/merge not authorized)
- commits: local commits on wi/0025-okf-registry in both repos; dai sha
  recorded in the wi-0025 handoff; the dai-vault closeout commit contains
  this file and cannot self-reference its sha (recorded in the operator
  slice report and via `git log wi/0025-okf-registry`)
- tests: none (docs-only; verification-command battery instead)
- verification notes: registry parses PS+py 17 keys; Test-Json true; strict
  snapshot 19 WIs / 0 continuations / 0 warnings; current-slice
  reconstruction proof passed; insertion-only diffs verified; staged sets
  equal allowlists (7 vault / 3 dai); protected baselines unchanged
- docs updated: the 10 allowlisted files above
- lessons: none (contract-driven doctrine slice; deferred observations in
  the handoff)

## final disposition

complete -- all 10 allowlisted files created or additively edited and
verified on 2026-07-16 per the reviewed authorization contract; O8 recorded
as retention; local commits on wi/0025-okf-registry in both repos; push and
merge remain unauthorized; Phase 2+ not begun; protected pre-existing state
untouched and re-verified against preflight baselines.

**Integration record (added 2026-07-24 completion audit).** The push/merge deferral above
was later lifted under separate authorization: the WI-0025 content is on published dai
main (`c6166e2` "docs(system): add WI-0025 okf registry schema and historical notices"
verified an ancestor of main 2026-07-24) and the coordinated vault content is on
published vault main. Phase 2+ remains not begun; that deferral is intact.
