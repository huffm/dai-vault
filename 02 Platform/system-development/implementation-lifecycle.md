---
title: "Implementation Lifecycle"
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
  - lifecycle
  - tdd
related:
  - "02 Platform/system-development/operating-model.md"
  - "02 Platform/system-development/work-item-traceability.md"
  - "06 Execution/patterns/agent-slice-workflow-doctrine-v1.md"
---

# implementation lifecycle

## purpose

The concrete stage sequence a work item moves through. Stages 5–7 are the existing slice
doctrine, unchanged; this doc adds the intake seam before it and the traceability seam after.

## stages

1. **Intake** — decide whether the change clears the meaningful-change threshold
   ([[operating-model]]). If yes, mint `WI-####` by creating the spec file from
   [[_template]].
2. **Spec** — fill the lite core at minimum: problem, acceptance criteria, test plan,
   verification commands, links block. Feature-class items fill all sections. The test plan
   is written BEFORE implementation — tests are part of the spec, not an afterthought.
3. **Review gate** — for feature-class items or anything touching contracts/doctrine, the
   spec is reviewed (by the user, or explicitly self-reviewed with the architecture-contracts
   lens) before code. Small fixes may proceed on the lite spec directly.
4. **Branch** — `wi/####-<slug>` per [[work-item-traceability]]. Direct-to-main is acceptable
   for docs-only work; code changes branch.
5. **Implement** — one or more slices, each run under `dai-slice-runner` (skills gate, bounded
   scope, TDD per `superpowers:test-driven-development`, targeted runs per
   `dai-test-discipline`). Pure logic is extracted into testable modules; UI work follows
   [[component-rules]] and [[interaction-states]].
6. **Verify** — execute the spec's verification commands verbatim; for UI work also run
   [[visual-qa-checklist]]. Evidence (output, screenshots) is referenced from the spec or the
   slice handoff, not pasted wholesale into the spec.
7. **Review** — code review before completion (`dai-code-reviewer` / `code-review`); blocking
   findings resolved before close.
8. **Close** — complete the links block (all 8 required links per [[work-item-traceability]]),
   record lessons in the spec, promote reusable lessons through the gate in
   [[operating-model]], update the docs named in "docs to update", append the slice handoff
   to `06 Execution/handoffs/current-slice.md`.

## definition of done

- [ ] acceptance criteria demonstrably met (verification evidence exists)
- [ ] test plan implemented; suite green; no skipped assertions without a recorded reason
- [ ] code review run; blocking findings resolved
- [ ] links block complete (8/8)
- [ ] docs-to-update list actioned
- [ ] lessons recorded (or explicit `none`)
- [ ] no changes to decision/prompt/calibration/reconciliation/confidence/artifact doctrine
      unless the spec explicitly scoped them

## what it is not

Not a replacement for the slice doctrine — stages 5–7 ARE the slice doctrine, referenced.
Not a gating bureaucracy: for a lite-tier item the whole lifecycle is spec (10 lines) →
implement → verify → close links.

## recommended next slice

Execute [[WI-0001-chip-primitive-and-long-token-treatment]] through all eight stages as the
proving run.
