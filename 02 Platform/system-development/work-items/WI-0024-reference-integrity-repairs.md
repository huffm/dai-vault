---
title: "WI-0024 Reference Integrity Repairs v1"
type: "plan"
date: "2026-07-16"
status: "complete"
project: "DAI"
slice: "WI-0024 Reference Integrity Repairs v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - reference-integrity
  - knowledge-system
related:
  - "02 Platform/system-development/operating-model.md"
  - "02 Platform/system-development/implementation-lifecycle.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# WI-0024 reference integrity repairs v1

## problem  <!-- LITE -->

Eleven active documents carry reference-integrity defects verified by the 2026-07-16
vault information-architecture audit: two dai docs end inside unclosed markdown fences
(`docs/request-flow.md` fence opened at line 8; `docs/agent-runs.md` fence opened at
line 20); four dai skill files plus one example doc cite pre-migration vault paths for
the slice-workflow doctrine and the prompt-ledger hook, which now live under
`06 Execution/patterns/`; and four vault docs carry broken front-matter `related:`
entries (two to the doctrine's old flat path, two to a nonexistent `ADR-0008-` filename
whose actual file is `02 Platform/decisions/0008-source-readiness-preflight-gate.md`).

## desired behavior

Every active skill, doc, and front-matter reference resolves to an existing file, so
agents and future indexing tooling can follow references deterministically.

## affected surfaces

dai: `docs/request-flow.md`, `docs/agent-runs.md`, `docs/examples/agent-paths.example.md`,
`.claude/skills/dai-slice-runner/SKILL.md`, `.claude/skills/dai-slice-prompt-architect/SKILL.md`,
`.claude/skills/dai-docs-architect/SKILL.md`, `.claude/skills/dai-agent-handoff/SKILL.md`.
dai-vault: `02 Platform/system-development/operating-model.md`,
`02 Platform/system-development/implementation-lifecycle.md`,
`02 Platform/system-development/work-items/WI-0005-starter-retrieval-caches-transport-failures.md`,
`02 Platform/system-development/work-items/WI-0006-identity-safe-mlb-doubleheader-resolution.md`.

## authority and design sources

- design source: DAI Knowledge System Architecture and Orchestration Plan v1.1, Phase 0
  (workspace root, outside both repos: `DAI-Knowledge-System-Architecture-v1.1.md`)
- execution scope: `WI-0024-Reference-Integrity-Repairs-Prompt.md` (workspace root);
  the narrower prompt controls on any conflict
- operator authorization: Phase 0 execution authorization prompt, 2026-07-16

## exact scope

11 repair files, 13 edit operations, exact-string matching only:
2 fence closures (append one closing fence line per file); 7 stale-path replacements
(`06 Execution/agent-slice-workflow-doctrine-v1.md` -> `06 Execution/patterns/...`,
2 in dai-slice-runner, 1 in dai-slice-prompt-architect;
`06 Execution/prompt-ledger-hook-v1.md` -> `06 Execution/patterns/...`,
1 each in dai-slice-prompt-architect, dai-docs-architect, dai-agent-handoff,
agent-paths.example.md); 4 related-path repairs (doctrine path in operating-model and
implementation-lifecycle; `ADR-0008-` -> `0008-` filename in WI-0005 and WI-0006).
Plus governance artifacts: this work item, the wi-0024 handoff, and the current-slice.md
closeout append under the exact-prefix preservation protocol.

## non-goals

No registry, banner, validator, views, Bases, Canvas, Codex-skill, or navigation work
(WI-0025 and later). No runtime, prompt, routing, schema, product, or test behavior
change. No file deletion, move, rename, or archive. No repair of stale references in
historical evidence bodies (deferred observations only). No push, no merge.

## protected pre-existing state

- dai `platform/dotnet/DevCore.Data/DevCore.Data.csproj`: modified (stat anomaly, empty
  diff), sha256 63EF248822D14E458C83C260BB6EC9E7CAEA842F345DD451F85F1B27C1D16A8F; never
  edited, staged, or reconciled.
- dai-vault `.obsidian/graph.json`: externally modified by Obsidian (obsidian running,
  file stable at execution baseline sha256
  59FD7ED1A5294C8ED64E9B228CE028EC9CBE15A8DFE9045104834781353F676A); never staged.
- dai-vault `06 Execution/handoffs/current-slice.md`: git-status modified but HEAD,
  index, and working-tree blob hashes are identical (ccbaff98 -- stat anomaly, no real
  content difference); prefix baseline 1383684 bytes, sha256
  866EB546DB7A3F11BF65A89B6BE5AF71C9C9010A51B2D2E910F580806E2DD75D; append-only.
- dai-vault untracked: `06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json`
  (sha256 68948EBD...) and `06 Execution/system-state-synopsis-v1.md` (sha256 25835E6C...);
  never modified, staged, or deleted.

## acceptance criteria  <!-- LITE -->

- Strict planning snapshot completes with zero warnings, output outside both repos.
- Both dai docs have an even fence-line count and render prose outside code blocks.
- Every `related:` entry in the four edited vault files resolves to an existing file.
- Zero occurrences of the two old path strings remain in `dai/.claude/skills/**/SKILL.md`
  and `dai/docs/examples/`; historical vault records intentionally still contain them.
- Baseline hashes of all four protected files unchanged at close (current-slice.md:
  baseline preserved as exact byte prefix with only the closeout entry appended).
- Staged sets match the explicit allowlists exactly; commits exist only on
  `wi/0024-reference-integrity`; nothing pushed or merged.

## test plan  <!-- LITE -->

Docs-only slice: no code tests. Verification is the command battery below plus
occurrence-count checks before and after each replacement (expected counts from the
2026-07-16 preflight: doctrine string 2+1 in dai skills, 1 each in two vault docs;
hook string 1 in each of four dai files; ADR string 1 in each of two vault docs).

## verification commands  <!-- LITE -->

- `pwsh dai/scripts/dev/planning/build-next-slice-snapshot.ps1 -OutputPath <scratch>/wi-0024-snapshot.json -Strict`
- fence-line count parity check on the two dai docs
- Test-Path resolution of every `related:` entry in the four edited vault files
- scoped grep for both old strings across `dai/.claude/skills/**/SKILL.md` and `dai/docs/examples/`
- `git diff --check` and full diff review in both repos
- sha256 re-hash of the four protected files against preflight baselines
- prefix-integrity hash of current-slice.md first 1383684 bytes after the append

## risks

Low. Text-only edits to non-parsed prose plus front-matter list entries that the
snapshot parser reads; mitigated by exact-string matching, pre/post occurrence counts,
and the strict snapshot re-run. Obsidian is running: graph.json is re-checked at every
gate and the slice stops if it drifts from the execution baseline.

## rollback

All changes are local commits on `wi/0024-reference-integrity` in both repos; rollback
is deleting the branches (or `git revert` of the commits if ever integrated). No
generated artifacts, no external effects, no pushes.

## links  <!-- LITE; all 8 required at close, per work-item-traceability -->

- work item: WI-0024
- branch: wi/0024-reference-integrity (dai and dai-vault)
- pr: none (push/merge not authorized)
- commits: local commits on wi/0024-reference-integrity in both repos; dai sha recorded
  in the wi-0024 handoff; the dai-vault closeout commit contains this file and cannot
  self-reference its sha (recorded in the operator slice report)
- tests: none (docs-only; verification-command battery instead)
- verification notes: strict snapshot 0 warnings (18 work items parsed, wi-0024
  included); fence parity 2/2 even; related-path resolution 12/12, 0 broken; scoped
  old-string audit 0 remaining; git diff --check clean both repos; changed-file sets
  equal the authorized scope exactly
- docs updated: the 11 repair files listed under affected surfaces; this work item;
  wi-0024 handoff; current-slice.md closeout append
- lessons: none (mechanical repair slice; deferred observations recorded in the handoff)

## final disposition

complete -- all 11 repair files / 13 edit operations applied and verified on 2026-07-16;
governance artifacts (this work item, the wi-0024 handoff, the current-slice.md append
under the exact-prefix protocol) created; local commits on wi/0024-reference-integrity
in both repos; push and merge remain unauthorized; no runtime, prompt, schema, or
product behavior changed; protected pre-existing state untouched and re-verified against
preflight baselines.

**Integration record (added 2026-07-24 completion audit).** The push/merge deferral above
was later lifted under separate authorization: the WI-0024 content is on published dai
main (`876b73a` "docs(system): repair WI-0024 reference integrity" verified an ancestor
of main 2026-07-24) and the coordinated vault content is on published vault main. The
"push and merge remain unauthorized" sentence describes the state at close only.
