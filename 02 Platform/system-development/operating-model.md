---
title: "DAI Development Operating Model"
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
  - operating-model
  - traceability
related:
  - "02 Platform/system-development/work-item-traceability.md"
  - "02 Platform/system-development/implementation-lifecycle.md"
  - "06 Execution/patterns/agent-slice-workflow-doctrine-v1.md"
---

# DAI development operating model

## purpose

Make every meaningful DAI change traceable end to end: a work item states the intent, an OKF
spec defines behavior and tests before code, implementation artifacts link back, and reusable
knowledge is recorded instead of dying in a session.

## the spine

Three artifact classes, each with one job. Nothing restates another; everything links.

| Artifact | Owns | Lives in |
|---|---|---|
| Work item (Azure DevOps; local `WI-####` until wired) | intent, state, links | ADO Boards (interim: the spec itself) |
| OKF spec | problem, acceptance criteria, test plan, lessons | `02 Platform/system-development/work-items/` |
| Code + tests | truth (values, behavior) | `dai` repo |

Docs cite code paths and never restate values. A doc that would need editing when a `.scss`
or `.cs` file changes is misdesigned.

## meaningful-change threshold

A work item is REQUIRED when a change alters behavior, a contract, UI, schema, or doctrine —
or spans more than one session. It is NOT required for typos, comments, doc hygiene, or
dependency bumps; those ride the nearest open work item or none. This rule is falsifiable on
purpose: if a change class keeps escaping it, amend the rule here rather than eroding it.

## layering (how this composes with existing process)

```
work item  (unit of intent -- may span several slices)
  └── slice (unit of execution -- runs dai-slice-runner unchanged: orient, bound,
             skills gate, execute, verify, review, handoff)
```

The system-development layer touches only two seams: intake/spec BEFORE the first slice and
traceability closeout AFTER the last. It coordinates existing skills and never re-implements
their steps. When this doc and `agent-slice-workflow-doctrine-v1.md` appear to conflict on
execution mechanics, the slice doctrine wins and this doc is corrected.

## lenses

Perspectives activated per task — prompt sections or ephemeral fan-out agents, never standing
personas with private memory. The main session is the owner: it selects lenses, arbitrates
conflicts, and is the only writer of canonical docs.

| Lens | Owns | Primary doc |
|---|---|---|
| frontend-implementation | Angular components, CSS architecture, tokens, responsive | [[frontend-implementation]] |
| backend-implementation | platform API, EF, endpoints, exposure gates | [[backend-implementation]] |
| architecture-contracts | truth hierarchy, DTO/API shape, buyer boundary, error semantics | [[architecture-contracts]] |
| testing-quality | test plans, regression traps, acceptance verification | [[testing-strategy]] |
| ui-system-design | visual/interaction doctrine, component rules, page critique | [[component-rules]], [[interaction-states]] |
| okf-curation | placement, front matter, links, MOC hygiene (applied at doc-writing time) | `06 Execution/patterns/okf-documentation-review-guide-v1.md` |

A lens splits only when its area outgrows one doc AND two consecutive work items needed the
sub-perspectives separately. Roles are added by evidence, not symmetry.

## learning loop

observe → critique → implement → test → record (spec's lessons section) → promote (canonical
doc) → link (OKF). Promotion gate, all required: reusable beyond one work item; no unresolved
conflict with existing doctrine; accepted by the owner; placed with correct front matter and
MOC linkage.

## what it is not

- Not a replacement for the slice doctrine, the skills gate, or any domain skill.
- Not a project-management methodology; there are no ceremonies, only artifacts and links.
- Not applicable to dai-vault strategy/calibration work, which keeps its own conventions.

## deferred decisions

- ADO org/project creation and Boards↔GitHub connection (see [[work-item-traceability]]).
- CI enforcement of commit trailers (manual convention until item volume justifies tooling).
- Per-lens doc splits.

## recommended next slice

Review and approve [[WI-0001-chip-primitive-and-long-token-treatment]], then execute it as
the proving run of this model.
