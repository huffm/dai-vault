---
title: "DAI Interaction States"
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
  - design-system
  - interaction
  - feedback
related:
  - "02 Platform/system-development/design-system/component-rules.md"
  - "02 Platform/system-development/architecture-contracts.md"
---

# dai interaction states

## purpose

Doctrine for async feedback: loading, success, empty, warning, error, and per-item states.
Documents the system that shipped in the dev-artifact-review revamp (dai `173d13d`,
`41c1c90`, `0119d7d`, 2026-07-09) so it becomes the pattern for every future surface instead
of a page-local invention.

## the status descriptor system

- One descriptor shape for all feedback: `{ variant: info|success|warning|error, title,
  detail }`, built by pure, unit-tested projection functions — never assembled inline in
  templates. Reference: `apps/sports-app/src/app/dev-artifact-review/review-feedback.ts`
  (+ spec).
- One rendering surface: a shared status banner component; errors render with
  `role="alert"` (assertive), everything else `role="status"` (polite). Reference:
  `dev-artifact-review/status-banner.ts`.
- Tone semantics are fixed page-wide (see [[component-rules]] R3): blue in-flight, green
  success, amber warning, red true error.

## per-item queue states

Multi-item async work models each item independently:
`pending → loading → loaded | unavailable | failed`.

- **Independence rule:** one item's failure never blocks or contaminates siblings; state
  lives on the item, keyed by its id. Reference: `dev-artifact-review/run-load.ts` (+ spec).
- **unavailable vs failed:** 404 = unavailable (warning tier, retryable, "may not have
  completed persistence"); other errors = failed (error tier, retryable). Mapping is a
  contract ([[architecture-contracts]]).
- Settled items are re-requestable: re-selecting a failed item re-attempts it; loaded items
  offer explicit reload.

## rules for every async surface

1. Every trigger has disabled + in-flight states; every input has a visible focus state.
2. Loading shows structure (skeleton or progress count), not a blank.
3. Empty states say what the user can do next — never a bare "no data".
4. Errors are contextual (in the section they belong to), carry the relevant identifier,
   and state the next action. A global summary may aggregate; it never replaces the local one.
5. Success is stated, not implied ("N runs loaded.", "Upcoming sample generation completed.").
6. Partial failure is a named state with counts ("2 of 3 runs loaded. 1 artifact
   unavailable. Loaded runs remain available below.").
7. Feedback copy lives in the pure projection module with tests — a copy change is a
   spec-covered change.

## what it is not

Not a streaming/concurrency doctrine (sequential processing is the current deliberate
choice for dev surfaces; a streaming doc splits out when a real streaming surface exists).
Not visual styling ([[component-rules]]).

## recommended next slice

Apply this doctrine to the next async surface built (it currently has one proving
implementation; the second consumer validates it as doctrine).
