---
title: "WI-0002 Artifact Chip Primitive Alignment"
type: "plan"
date: "2026-07-09"
status: "blocked"
project: "DAI"
slice: "WI-0002 artifact chip primitive alignment"
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
  - "02 Platform/system-development/design-system/component-rules.md"
---

# WI-0002 artifact chip primitive alignment

**Status: BACKLOG. Implementation authorization: NOT AUTHORIZED.**
Registered 2026-07-09 from a WI-0001 deferred candidate. This spec records the decision
space; it does not pre-decide the outcome. Nothing in it is doctrine yet.

## problem statement

The dev-artifact-review page carries two pill-adjacent primitives: the shared `.chip`
system (WI-0001) and the local `.artifact-chip` squared signal-coverage tags
(`--grounded`/`--missing`, `dev-artifact-review.component.scss`). Two primitives on one
page is a standing divergence risk: geometry or tone decisions made on `.chip` do not
automatically reach `.artifact-chip`.

## current evidence

- `.artifact-chip` is presently a **distinct squared tag shape** (0.5rem radius, different
  padding), used at exactly two template sites (grounded/missing signal chips).
- It is **not an incomplete WI-0001 migration** — the WI-0001 final review verified this
  and the deferral was deliberate (see WI-0001 "follow-up items" and the corrected SCSS
  comment).
- It already consumes the shared `--status-*` tokens (WI-0001, R3), so tone drift is
  already prevented; only geometry/structure remain unshared.

## business / maintenance rationale

One primitive is cheaper to evolve than two — but only if the two are actually the same
thing. If the squared shape carries intentional meaning (coverage tags vs status pills),
forcing them into one primitive destroys information to save ~20 lines of CSS.

## scope

Inventory and design decision first; mutation only if the decision requires it, and only
within dev-artifact-review styles/templates.

## non-goals

No new visual design; no page redesign; no backend/contract/doctrine change; no changes to
what the tags mean; no shared-module extraction (that is WI-0003's question).

## activation gate

Implement only when one of: (a) a chip-primitive change lands that `.artifact-chip` should
have received but did not (observed drift), or (b) a design pass touches the
signal-coverage tags anyway, or (c) a second surface needs squared tags.

## required inventory before any mutation

Enumerate every `.artifact-chip` usage, its semantic role, and the visual deltas vs
`.chip--status`. Then choose a disposition — all four remain open:

1. retain `.artifact-chip` as a distinct primitive (document the distinction as intended)
2. share tokens only (current state, made explicit)
3. share structural rules with a shape modifier (e.g. `.chip--tag` square variant)
4. migrate fully to `.chip` when evidence supports it

## acceptance criteria (for the future implementation slice)

1. A recorded disposition with rationale, before any CSS change.
2. If mutated: zero visual regression on grounded/missing tags at 390/768/1440 px,
   verified against before screenshots.
3. Cascade order rules of component-rules R1 respected (modifiers after tones).
4. Grep proof matching the chosen disposition (e.g. `.artifact-chip` gone, or documented).

## required lifecycle stages

All eight stages of [[implementation-lifecycle]]; spec review gate applies (feature-class:
it touches a shared primitive).

## required links (at close)

The standard 8 links per [[work-item-traceability]].

## verification expectations

Frontend suite green; visual QA checklist at all three widths; computed-style checks for
any combination the change introduces (WI-0001 lesson 5: source order is the API).

## responsive and accessibility expectations

No new contrast regressions (tokens already AA-verified in WI-0001 review); tags remain
text-bearing (never color-only).

## risks

- Over-alignment: erasing an intentional visual distinction between coverage tags and
  status pills.
- Under-alignment: leaving a second primitive that silently misses the next chip evolution.

## dependencies

WI-0001 (closed, integrated at dai `bb10c3c`).

## rollback / containment posture

CSS + template-class change only; revert = single commit revert; no data or contract
surface.

## definition of done

Per [[implementation-lifecycle]] plus the recorded-disposition requirement above.

## relationship to WI-0001

Direct descendant: WI-0001 built the `.chip` primitive and deliberately left
`.artifact-chip` out after verifying it is a distinct shape, not a missed migration.
