---
title: "MOC - DAI System Development"
type: "moc"
date: "2026-07-09"
status: "in-progress"
project: "DAI"
slice: "DAI System Development v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - system-development
  - moc
  - okf
related:
  - "02 Platform/system-development/operating-model.md"
  - "06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md"
---

# MOC - DAI System Development

Hub for the DAI development operating model: how work is specified, traced, implemented,
tested, and turned into reusable knowledge. Execution mechanics stay in the slice doctrine
(`06 Execution/agent-slice-workflow-doctrine-v1.md`); this subtree governs intent and
traceability around it.

## front matter placement decision (recorded here on purpose)

The canonical 9-field OKF schema (`06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md`)
is documented for `06 Execution` docs. This subtree deliberately extends the same schema —
same nine fields, same allowed values — to `02 Platform/system-development/`. One new `type`
value is introduced: `moc` (no existing value fits a map-of-content). This is a placement
extension, not a second dialect. A future hygiene audit should read this as intent, not drift.

## operating model

- [[operating-model]] — the spine (work item / spec / code), meaningful-change threshold, six lenses, learning loop
- [[work-item-traceability]] — IDs, naming, link topology, local-spine mode until Azure DevOps is wired
- [[implementation-lifecycle]] — intake → spec → gate → branch → implement → verify → review → close

## implementation doctrine

- [[frontend-implementation]] — Angular/sports-app repo patterns (seeded, in-progress)
- [[backend-implementation]] — platform API repo patterns (seeded, in-progress)
- [[architecture-contracts]] — truth hierarchy, buyer boundary, correlation, error semantics
- [[testing-strategy]] — what must be tested per change class

## design system

- [[component-rules]] — chip primitive, long-token treatment, status color tokens
- [[interaction-states]] — the shipped status/feedback system (StatusDescriptor, queue states)
- [[visual-qa-checklist]] — pre-completion visual gate

## work items

- [[_template]] — tiered work-item spec template (lite core + feature-class sections)
- [[WI-0001-chip-primitive-and-long-token-treatment]] — chip primitive, long-token treatment,
  status tokens (**complete** 2026-07-09; first proving run of the model, branch
  `wi/0001-chip-primitive`, unmerged)

## skill layer

- `dai/.claude/skills/dai-system-development/SKILL.md` — the workflow router (coordinates
  existing skills; never replaces them). Tracked in
  `06 Execution/skills/dai-skills-inventory-v1.md`.
