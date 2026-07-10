---
title: "WI-0003 Shared Chip and Long-Token Module Promotion"
type: "plan"
date: "2026-07-09"
status: "blocked"
project: "DAI"
slice: "WI-0003 shared chip and long-token module promotion"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - design-system
  - backlog
related:
  - "02 Platform/system-development/work-items/WI-0001-chip-primitive-and-long-token-treatment.md"
  - "02 Platform/system-development/frontend-implementation.md"
---

# WI-0003 shared chip and long-token module promotion

**Status: BACKLOG. Implementation authorization: NOT AUTHORIZED.**
Registered 2026-07-09 from a WI-0001 deferred candidate. This spec records the activation
gate; it does not preselect an architecture. Nothing in it is doctrine yet.

## problem statement

`.chip` (global classes in `apps/sports-app/src/styles.css`) and `app-long-token`
(component in `app/dev-artifact-review/long-token.ts`) are design-system primitives whose
only page-level consumer is dev-artifact-review. When a second surface needs them, their
current ownership (page folder for the component; global stylesheet for the classes) will
be the wrong boundary — but promoting them before that demand exists is speculative
abstraction.

## current evidence

- Exactly one page-level consumer (verified in the WI-0001 final review: `chip--` matches
  only under `dev-artifact-review/`).
- `app-long-token` lives in the dev page's folder with its spec beside it, per the repo's
  pure-module pattern.
- WI-0001's frontend-implementation pattern 7 explicitly records this as "a pattern with
  one page-level consumer, not established doctrine."

## business / maintenance rationale

Premature extraction creates an ownership boundary shaped by one consumer's needs; the
second consumer then bends around it. Extraction on real demand shapes the boundary from
two data points.

## scope (future slice, gated)

Move `.chip` documentation/ownership and `app-long-token` into an explicitly shared
boundary sized to the actual second consumer.

## non-goals

No new features on either primitive; no visual changes; no NgModule for its own sake; no
extraction of the spinner or other single-consumer helpers "while we're in there".

## activation gate (all required before implementation)

1. Do NOT implement while dev-artifact-review remains the only page-level consumer.
2. A concrete second consumer is identified by name (page/component + the specific need).
3. The second consumer's requirements are verified materially compatible (size/tone set,
   token behavior, copy affordance semantics) — not assumed.
4. If the second use is not sufficiently similar, record "no extraction" as the outcome
   and close the item; a third candidate reopens it.

## candidate implementation forms (none preselected)

- shared standalone component ownership (e.g. `app/shared/ui/long-token.ts`)
- shared styles ownership only (chips stay CSS; component stays local)
- feature-shared library folder if more shared UI accumulates
- no extraction (documented incompatibility)

## acceptance criteria (for the future implementation slice)

1. Activation gate demonstrably satisfied and recorded in the spec before code moves.
2. Both consumers render from the shared boundary with zero visual regression
   (before/after screenshots at 390/768/1440).
3. `long-token.spec.ts` moves with the module and stays green; suite green.
4. Import paths updated everywhere; no duplicate copies remain (grep proof).

## required lifecycle stages

All eight stages of [[implementation-lifecycle]]; feature-class spec review applies.

## required links (at close)

The standard 8 links per [[work-item-traceability]].

## verification expectations

Full frontend suite; focused long-token spec; visual QA checklist on BOTH consumers;
computed-style spot checks for chip combinations on the new surface.

## responsive and accessibility expectations

The new consumer inherits the WI-0001 guarantees: AA contrast tones, `<wbr>` separator
wrapping with `overflow-wrap: anywhere` fallback, copy affordance with visible focus.

## risks

- Boundary shaped too early (mitigated by the gate).
- Silent divergence if a second consumer copies instead of promoting — watch for
  copy-paste of `.chip` rules or `splitToken` as the actual trigger signal.

## dependencies

WI-0001 (closed, integrated at dai `bb10c3c`); a real second consumer (does not exist yet).

## rollback / containment posture

File moves + import updates; revert = single commit revert; no data or contract surface.

## definition of done

Per [[implementation-lifecycle]] plus the activation-gate record above.

## relationship to WI-0001

Direct descendant: WI-0001 created both primitives and deliberately left them at page
scope pending a second consumer.
