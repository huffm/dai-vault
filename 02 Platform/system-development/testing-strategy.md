---
title: "Testing Strategy"
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
  - testing
  - tdd
related:
  - "02 Platform/system-development/implementation-lifecycle.md"
  - "02 Platform/system-development/design-system/visual-qa-checklist.md"
---

# testing strategy

## purpose

What must be tested per change class. The `dai-test-discipline` skill owns HOW to run tests
cheaply (targeted runs, red/green cadence); `superpowers:test-driven-development` owns the
write-test-first mechanics. This doc owns the WHAT.

## rules

1. **Tests are defined in the spec before implementation.** A work item's test plan is part
   of stage 2 (spec), not stage 5 (implement). Code written before its failing test violates
   the lifecycle.

2. **Per change class:**

| Change class | Required tests | Mechanism |
|---|---|---|
| Pure logic (parsing, state transitions, projections) | unit specs, TDD | vitest, spec file beside the module |
| Angular component behavior | extract the logic into a pure module and test that; TestBed only when extraction is genuinely impossible | vitest |
| Visual/UI change | rendered pass at 390/768/1440 px covering loading/empty/error/success/partial states | [[visual-qa-checklist]] + dev server |
| API/contract change | endpoint tests in the platform test suite; DTO mirror updated with the same item | `dotnet test` (platform suites) |
| Doctrine-adjacent change (should not happen from this subtree) | full platform + frontend suites green as regression proof | both suites |

3. **Partial-failure behavior is a first-class test target.** Any multi-item async surface
   tests: one item failing does not block siblings; per-item status transitions; feedback
   states for empty/partial/total failure. Precedent: `run-load.spec.ts`,
   `review-feedback.spec.ts` (40 tests covering exactly this class, 2026-07-09).

4. **Suites stay green at every close.** Frontend baseline: `npx ng test --watch=false`
   (130 tests as of 2026-07-09). A work item that leaves a suite red cannot close, per the
   definition of done in [[implementation-lifecycle]].

5. **Verification is command-based.** The spec lists verification commands; stage 6 runs them
   verbatim and records outcomes. "It works" without command output is not verification
   (`superpowers:verification-before-completion`).

## what it is not

Not a test-runner guide (dai-test-discipline), not TDD mechanics (the superpowers skill),
not a coverage-percentage policy — coverage targets are deliberately absent until evidence
shows a gap class that line-coverage would have caught.

## deferred decisions

- Visual regression tooling (screenshot diffing) — revisit if manual visual QA misses twice.
- Backend test conventions doc — with the first backend-touching work item.

## recommended next slice

None standalone; WI-0001's test plan is the first consumer.
