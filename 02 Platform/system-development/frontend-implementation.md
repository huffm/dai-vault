---
title: "Frontend Implementation Patterns"
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
  - frontend
  - angular
related:
  - "02 Platform/system-development/design-system/component-rules.md"
  - "02 Platform/system-development/testing-strategy.md"
---

# frontend implementation patterns

## purpose

Repo-specific patterns for `dai/apps/sports-app` (Angular 21, signals, Tailwind 4 + SCSS,
vitest). Seeded from verified shipped code; grows by promotion from work-item lessons. The
`dai-typescript-angular-quality` skill owns in-session type discipline; this doc owns the
repo's structural patterns.

## patterns (each verified in shipped code)

1. **Standalone components + signals.** Components are standalone; state is `signal()`,
   projections are `computed()`. Derivable state is never a second signal.
   Reference: `apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.ts`.

2. **Pure logic modules beside the component, spec-first.** Non-trivial logic (parsing, state
   transitions, feedback projection) lives in a pure module with a vitest spec next to it;
   the component orchestrates. This is the repo's dominant testing seam — prefer it over
   TestBed component tests.
   References: `dev-artifact-review/run-load.ts`, `review-feedback.ts`, `signal-table.ts`,
   `run-anatomy.ts` (each with a sibling `.spec.ts`).

3. **Tokens live in one file.** Color/spacing/effect tokens are CSS custom properties in
   `apps/sports-app/src/styles.css` (`--app-*` surfaces/text, `--status-*` semantic tones),
   exposed to Tailwind via `@theme inline`. New visual values start as tokens there, not as
   literals in component styles. Status tones are consumed as `rgba(var(--status-X-rgb), a)`.
   (WI-0001 resolved the prior three-way duplication; see [[component-rules]] R3.)

4. **Utility classes for layout, component SCSS for primitives.** Tailwind utilities handle
   layout/spacing in templates; reusable visual primitives (`.artifact-status`,
   `.queue-row`, `.status-banner`) get classes in component SCSS or shared styles.

5. **Dev tooling is fail-closed.** Dev-only surfaces gate through pure predicates over
   environment flags (`core/dev-tools.ts`); paid-call entry points require their own flag in
   addition to the route gate. Never weaken these in a UI slice.
   Reference: `docs/sports-exposure-boundary.md` in the dai repo.

6. **Buyer/dev boundary is server-side.** Buyer surfaces consume buyer projections
   (`getBuyerAgentRunArtifact`); internal fields are dropped by the API, never hidden
   client-side. See [[architecture-contracts]].

7. **Shared visual primitives live in `styles.css`, not in components.** The `.chip`
   primitive (sizes, tones, modifiers) is global because it is consumed by many sections of
   a page and by encapsulated components (`status-banner`) that cannot see another
   component's styles; a consumer composes classes rather than restyling one. Today the only
   page consuming it is dev-artifact-review — this is a pattern with one page-level consumer,
   not established doctrine (see the WI-0003 candidate). Promoted from WI-0001.

8. **Unlayered component classes beat Tailwind utilities.** Tailwind v4 emits utilities into
   `@layer utilities`; plain CSS in `styles.css` is unlayered and therefore wins regardless
   of specificity or order. A component class that sets `color` (e.g. `.chip--badge`)
   silently ignores `text-*` utilities on the same element — expose a modifier instead
   (`.chip--quiet`). Verified by computed style, not by reading the CSS. Promoted from
   WI-0001 code review.

## what it is not

Not a style guide for visuals ([[component-rules]] and [[interaction-states]] own that), and
not a testing doc ([[testing-strategy]]). Not generic Angular advice — every pattern here
must cite a dai file.

## deferred decisions

- Shared spinner component extraction (still one consumer).
- Moving `.chip` / `app-long-token` into a shared UI module once a second surface uses them
  (candidate WI-0003).
- Folding `.artifact-chip` (squared tags) into the chip primitive (candidate WI-0002).
- Component-test strategy if a future surface can't extract its logic into pure modules.

## recommended next slice

WI-0002 or WI-0003 above, when a second surface creates real demand.
