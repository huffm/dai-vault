---
title: "Work Item Traceability"
type: "execution-pattern"
date: "2026-07-09"
status: "in-progress"
project: "DAI"
slice: "DAI System Development v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - traceability
  - azure-devops
related:
  - "02 Platform/system-development/operating-model.md"
  - "02 Platform/system-development/work-items/_template.md"
---

# work item traceability

## purpose

One set of naming and linking conventions so any artifact (spec, branch, PR, commit, test,
doc, lesson) can be traced to its work item and back — working from day one, before Azure
DevOps is connected, and unchanged after.

## current reality (verified 2026-07-09)

Both repos are GitHub-hosted (`huffm/devcore-ai`, `huffm/dai-vault`). No Azure DevOps org,
project, or Boards↔GitHub connection exists in this workspace yet. Therefore v1 runs in
**local-spine mode**: work-item IDs are minted in the vault and the spec doubles as the
work item's state holder.

## identifiers

- Local mode: `WI-####`, zero-padded, minted sequentially by creating the spec file. The
  spec filename is the registry — grep `work-items/` for the highest number.
- ADO mode (after wiring): the ADO work item ID is authoritative; `AB#<id>` appears alongside
  the local ID in the spec's links block. Existing `WI-####` IDs are never renumbered.

## naming conventions

| Artifact | Convention | Example |
|---|---|---|
| Spec file | `work-items/WI-####-<slug>.md` | `WI-0001-chip-primitive-and-long-token-treatment.md` |
| Branch | `wi/####-<slug>` | `wi/0001-chip-primitive` |
| Commit | trailer line `WI: WI-####` (ADO mode: also `AB#<id>` for auto-linking) | `fix(sports): chip primitive\n\nWI: WI-0001` |
| PR | title prefix `[WI-####]`, description links the spec path | `[WI-0001] chip primitive + long-token treatment` |

## link topology and single-writer rules

```
work item (state, links)  <->  spec (content)  <->  code (truth)
```

- The work item (ADO; interim: the spec's status field + links block) owns STATE and LINKS.
- The spec owns CONTENT: problem, desired behavior, acceptance criteria, test plan,
  verification notes, lessons.
- Code owns TRUTH. Specs cite paths; they never restate values or duplicate diffs.
- Nothing is written in two places. A broken link is a defect; a duplicated paragraph is a
  worse one.

## required links at close

A work item may be closed only when its spec's links block contains:

1. work item ID (and ADO URL once wired)
2. branch name
3. PR URL (or "merged direct: <commit range>" for solo-repo direct commits)
4. commit hashes
5. tests added/updated (paths)
6. verification notes (commands run + outcomes, or a pointer to the slice handoff)
7. docs created/updated (vault paths)
8. reusable lessons recorded (or explicit `none`)

## ado adoption path (deferred, no convention changes)

1. Create ADO org + project; choose process template (Basic is sufficient).
2. Install the Azure Boards app on the GitHub org/repos; `AB#<id>` in commits/PRs then
   auto-links.
3. New items get ADO IDs; each spec's links block records both IDs. Local mode remains the
   documented fallback for offline work.

## what it is not

Not an issue tracker workflow (no swimlanes, sprints, or estimates in v1). Not retroactive:
pre-WI-0001 history is not backfilled.

## recommended next slice

Wire ADO when item volume makes the local registry awkward (roughly >20 open items), not before.
