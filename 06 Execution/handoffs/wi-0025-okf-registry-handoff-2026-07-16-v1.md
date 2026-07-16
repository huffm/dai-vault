---
title: "WI-0025 OKF Registry and Authority Contract v1 -- slice handoff (2026-07-16)"
type: "handoff"
date: "2026-07-16"
status: "complete"
project: "DAI"
slice: "WI-0025 OKF Registry and Authority Contract v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - system-development
  - okf
  - registry
  - knowledge-system
  - handoff
related:
  - "02 Platform/system-development/work-items/WI-0025-okf-registry-and-authority-contract.md"
  - "06 Execution/patterns/okf-registry-v1.md"
  - "02 Platform/system-development/knowledge-system.md"
---

# HANDOFF: WI-0025 OKF Registry and Authority Contract v1

Design source: DAI Knowledge System Architecture and Orchestration Plan v1.1, Phase 1
(workspace root, outside both repos), as corrected by the reviewed authorization contract
`WI-0025-OKF-Registry-and-Authority-Contract-Authorization-Prompt.md` (workspace root;
sha256 51B04FEA7EE288004B90290D51B2B3E1F8F5BF823232DC228F9EFA43965102FC, 32338 bytes,
629 lines). Operator authorization: WI-0025 Phase 1 execution authorization, 2026-07-16.

## work-item identity (WI-0007)

- governing work item: WI-0025 OKF Registry and Authority Contract v1
- work-item status: complete (front matter); body disposition: complete, local commits
  only, push/merge not authorized
- okf path: `02 Platform/system-development/work-items/WI-0025-okf-registry-and-authority-contract.md`
- moc disposition: unchanged -- MOC - Execution Records does not exist until Phase 3
  (WI-0027); no MOC is in this slice's allowlist
- implementation/integration state: implementation complete on wi/0025-okf-registry
  branches in both repos; integration (push/merge) separately gated and not authorized

## 1. Objective

Phase 1 of the knowledge-system plan: create the OKF registry and authority contract
(registry doctrine + JSON Schema + knowledge-system living contract), add the D4/D5/D6
additive notices to one rolling and four historical documents, record the O8 retention
disposition, and mint the WI-0025 governance artifacts. Ten allowlisted files, two local
commits, nothing else.

## 2. Outcome

All 10 allowlisted files created or additively edited and verified. Created (5):
`06 Execution/patterns/okf-registry-v1.md` (single fenced JSON block, 17 top-level keys,
type_folders as canonical-parent arrays, both legacy date gates, generated-artifact
lifecycle), `02 Platform/system-development/knowledge-system.md` (living contract, O5
rationale + revisit trigger, graph-filter documentation, pending operator decisions),
`dai/scripts/knowledge/schemas/okf-registry.schema.json` (draft 2020-12), the WI-0025
work item, this handoff. Additively edited (5): D4 banner + closeout append in
`current-slice.md`; D5 banners in `dai/docs/current-state.md`,
`dai/docs/session-handoff.md`, `06 Execution/prompting/next-slice.md`; D6 errata in
`06 Execution/skills/dai-skills-inventory-v1.md`. All five controlling review
corrections applied. O8 disposition: the untracked synopsis is retained untouched;
cleanup requires separate authorization.

## 3. Repo State

### Before
- `dai`: main, 876b73aa3be678b9e20fbe71e8f5d442deb424b6, in sync with origin/main;
  modified (protected, stat anomaly, empty diff): platform/dotnet/DevCore.Data/DevCore.Data.csproj;
  nothing staged
- `dai-vault`: main, 014469fecc8b3a3da42c6ecdc0bc0708e9d81663, in sync with origin/main;
  modified (protected): .obsidian/graph.json (obsidian closed, hash stable
  2313F26F836E156EB2813506C67D7AC499E76AC7A12EC8456F011EDCB643884A); untracked
  (protected): 06 Execution/reports/preflight-settlement-manifest-2026-07-06-v1.json,
  06 Execution/system-state-synopsis-v1.md; current-slice.md git-clean at baseline
  5F2A3FEE95A28301D9A173574CEF1D76BC3CA70E8C4919C24321F0102601252B (1400653 bytes)

### After
- `dai`: wi/0025-okf-registry, local commit c6166e2de9238b4109beb6a975fd2f830447ef13
  (3 files, +169 insertions, 0 deletions), not pushed, no upstream; csproj still
  modified and unstaged (unchanged from baseline)
- `dai-vault`: wi/0025-okf-registry, closeout commit contains the 7 allowlisted vault
  paths including this handoff (the commit sha cannot be self-referenced here; it is
  recorded in the operator slice report and via `git log wi/0025-okf-registry`);
  protected files untouched and unstaged; current-slice.md final state
  B94BBA279382793DDA0631E1D634C105C8EBD53E748658A382EB95F2B8D6FD3A (1403460 bytes)

## 4. Services Used

- none: no runtime, database, model, or external service was touched; docs-only

## 5. Work Performed

- controlling-contract integrity check (sha256/bytes/lines matched the authorization)
- read-only preflight, all 10 steps: repo SHAs as expected; protected baselines;
  no unexpected dirty/untracked state; obsidian closed with graph.json stable across
  two immediate reads; current-slice baseline verified; creation targets absent, edit
  targets present; mint check (highest = WI-0024); strict snapshot 18/0/6/5/0; OKF
  type/folder inventory (19 execution-pattern = 10 platform-side + 9 patterns-side;
  24 noncanonical active-type docs, all dated before 2026-07-16, none on/after)
- branches wi/0025-okf-registry created in both repos from the expected mains
- 5 files authored; 5 files additively spliced at byte level (per-file EOL preserved:
  CRLF for current-slice.md, LF elsewhere; no BOM introduced; final-newline state
  preserved everywhere)
- full verification gate run (section 9); explicit-allowlist staging; local commits

## 6. Files Changed

- `dai/scripts/knowledge/schemas/okf-registry.schema.json` -- created (schema for the registry block)
- `dai/docs/current-state.md` -- D5 historical banner inserted under the first heading
- `dai/docs/session-handoff.md` -- D5 historical banner inserted under the first heading
- `dai-vault/06 Execution/patterns/okf-registry-v1.md` -- created (registry doctrine + JSON block)
- `dai-vault/02 Platform/system-development/knowledge-system.md` -- created (living contract)
- `dai-vault/02 Platform/system-development/work-items/WI-0025-okf-registry-and-authority-contract.md` -- created
- `dai-vault/06 Execution/handoffs/current-slice.md` -- D4 banner inserted + closeout appended
- `dai-vault/06 Execution/prompting/next-slice.md` -- D5 historical banner inserted
- `dai-vault/06 Execution/skills/dai-skills-inventory-v1.md` -- D6 errata footnote appended
- `dai-vault/06 Execution/handoffs/wi-0025-okf-registry-handoff-2026-07-16-v1.md` -- this handoff (created)

## 7. DB Writes / External Side Effects

- none

## 8. Paid Calls / Cost

- paid model calls: 0
- estimated cost: 0
- proof: docs-only slice; no service, endpoint, or model was invoked

## 9. Validation Proof

- registry parsing gates: PowerShell ConvertFrom-Json ok (17 top-level keys,
  schema_version 1.0, exactly 1 fenced json block in the document); Python json.loads
  ok (17 top-level keys); registry semantic checks ok (execution-pattern accepts all
  three canonical parents; both legacy date gates present and equal 2026-07-16;
  path_rules.type_folder_match = exact-parent-directory; all four generated artifacts
  planned with delivery WIs {current-state-view/execution-index/operating-model-canvas:
  WI-0027, corpus: WI-0026})
- Test-Json against the checked-in draft 2020-12 schema: True; negative controls
  (invalid state enum; removed legacy_type_folder_ok_before) both correctly rejected;
  schema file itself parses in both PowerShell and Python
- strict planning snapshot (scratch output, outside both repos): 19 work items
  (WI-0025 added; baseline 18), 0 continuations, 6 deferred candidates (unchanged),
  5 timeline entries (unchanged), 0 warnings -- the only delta vs baseline is the
  minted work item; re-run after this handoff was written confirmed identical counts
- additive integrity: current-slice.md final-minus-insertions reproduces baseline
  sha256 5F2A3FEE95A28301D9A173574CEF1D76BC3CA70E8C4919C24321F0102601252B byte-for-byte;
  byte-reconstruction proofs passed for all five edited files; numstat insertions-only
  (current-slice +34/0, next-slice +2/0, inventory +2/0, current-state +2/0,
  session-handoff +2/0)
- banner path check: D4/D5 texts reference only `06 Execution/handoffs/` and the strict
  snapshot; no reference to the not-yet-created 06 Execution/state/current-state-view.md
- related-path resolution: every `related:` entry in the four new OKF documents resolves
- `git diff --check`: clean in both repos
- staged sets equaled the allowlists exactly: dai 3 paths, dai-vault 7 paths
- protected baselines at close: csproj 63EF2488... unchanged; graph.json 2313F26F...
  stable at every gate; both untracked files unchanged (68948EBD..., 25835E6C...);
  none staged or committed
- no persistent assistant memory was written, updated, or deleted during execution

## 10. What Did Not Change

- prompts: unchanged
- routing: unchanged
- confidence logic: unchanged
- buyer copy: unchanged
- migrations/schema: unchanged (the new okf-registry.schema.json is a docs-layer
  contract file, not a database or API schema)
- runtime behavior: unchanged (no application, platform, service, or test file touched)

## 11. Open Issues

- none for this slice. Deferred by design: Welcome.md disposition (Phase 3, operator
  approval); untracked synopsis cleanup (separate authorization; O8 = retention);
  historical old-path references in evidence bodies remain per the D6 errata model;
  Phase 2+ tooling entirely unstarted.

## 12. Recommended Next Slice

WI-0025 integration (publish + ff both mains) under a separate operator authorization,
then Phase 2: WI-0026 parser extraction, Invoke-DaiKnowledge.ps1
(status/audit/validate/build), manifest + golden tests -- pending its own authorization
prompt.

## 13. Suggested Next Prompt

```text
Integrate WI-0025 (wi/0025-okf-registry, dai c6166e2 + the dai-vault closeout commit)
under the same integration contract as WI-0024: independent re-verification, protected
dirty-state preservation, topic-branch publication, ff-only main advancement, no force.
Then prepare (do not execute) the WI-0026 Phase 2 authorization prompt from
DAI-Knowledge-System-Architecture-v1.1.md section 13 and the integrated registry.
```

### Slice Synopsis

**Change:** WI-0025 shipped the OKF registry and authority contract: registry doctrine
with a single 17-key JSON block, its draft 2020-12 schema, the knowledge-system living
contract, D4/D5/D6 additive notices on five historical/rolling documents, and the O8
retention record.
**Reason:** the vault's metadata convention had no machine-readable contract, and stale
current-state documents still presented themselves as live truth.
**Proof:** registry parses in PowerShell and Python; Test-Json true with negative
controls rejected; strict snapshot 19/0/6/5/0 warnings; current-slice baseline
reproduced byte-for-byte minus insertions; insertion-only diffs on all five edits.
**State:** local commits on wi/0025-okf-registry in both repos (dai c6166e2); nothing
pushed or merged; protected dirty state preserved.
**Next:** WI-0025 integration -- awaiting separate operator authorization.
