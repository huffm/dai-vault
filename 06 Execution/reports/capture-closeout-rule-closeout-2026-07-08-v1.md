---
title: "Capture-Closeout Rule v1 -- closeout report"
type: "evidence-report"
date: "2026-07-08"
status: "COMPLETE -- docs-only; rule active"
project: "DAI"
slice: "Capture-Closeout Rule v1"
repos:
  dai: "unchanged (a0db824)"
  dai-vault: "docs-only"
related:
  - "06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md"
  - "06 Execution/reports/run-identity-hygiene-v2-2026-07-08-v1.md"
---

# capture-closeout rule v1 -- closeout

## objective

docs-only process rule: evidence/QA/soak/canary/diagnostic reruns of real games must
be excluded (diagnostic/superseded/invalid) at creation, or at slice closeout at the
latest, so they cannot contaminate identity settlement, calibration denominators, or
SingleMatch selection.

## delivered

- pattern doc (born OKF-compliant, type execution-pattern):
  `06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md` -- recurrence
  risk (22 retro-exclusions across 19 gamePks in hygiene v1+v2), run-type -> reason
  mapping (diagnostic / superseded / invalid), timing (creation preferred, closeout
  hard deadline, postponement sub-rule from 823613), and the required closeout
  evidence: per-run eligibility list + zero-row duplicate-active sweep (sql included),
  with non-zero sweep = closeout BLOCKER.
- current-slice.md append referencing the rule (this slice's entry).

## definition-of-done check

- future capture/canary/QA/soak runs have a documented closeout obligation: YES
  (pattern doc sections 2-4; binding status in front matter).
- rule explicitly protects SingleMatch safety + calibration denominator integrity:
  YES (section 4 closing paragraph, by-construction argument).
- dai untouched: YES (a0db824; no code/schema/prompt/model/reconcile/capture change).
- vault commit created and ready to push: YES (this commit).

## baseline at rule activation

duplicate-active gamePks = 0 (hygiene v2 end state, verified 2026-07-08); excluded
runs 38; valid-settled 121. the rule's sweep obligation starts from a clean surface.
