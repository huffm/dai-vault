---
title: "DAI Visual QA Checklist"
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
  - qa
  - accessibility
related:
  - "02 Platform/system-development/design-system/interaction-states.md"
  - "02 Platform/system-development/testing-strategy.md"
---

# dai visual qa checklist

## purpose

The pre-completion gate for any UI-touching work item. Run against the real rendered app
(dev server), not build output or screenshots from memory. Accessibility checks live here
until they outgrow this doc.

## widths

- [ ] 390 px — no horizontal scroll; rows/cards wrap intentionally; tap targets usable
- [ ] 768 px — intermediate layout sane (stacked vs side-by-side transitions clean)
- [ ] 1440 px — measure/line-length acceptable; no orphaned controls

## states (per async surface on the page)

- [ ] loading (skeleton/progress visible, triggers disabled)
- [ ] empty (says what to do next)
- [ ] success (stated, correct counts)
- [ ] warning/partial failure (counts correct; siblings unaffected)
- [ ] error (contextual, carries identifier, retry path works)

## components

- [ ] chips/badges: single primitive, aligned text, no optical off-center (R1)
- [ ] long tokens: no mid-token shatter; copy affordance present (R2)
- [ ] status tones: correct semantics — green success only, amber warning only, red error
      only, blue in-flight only (R3)

## accessibility

- [ ] keyboard: full flow operable without a mouse; focus visible on every interactive element
- [ ] error banners announce (`role="alert"`); polite regions for the rest
- [ ] labels: every input labeled; icon-only buttons have `aria-label`
- [ ] contrast: text on tinted/dark surfaces spot-checked (amber/muted text on dark is the
      known risk area)
- [ ] reduced motion: `prefers-reduced-motion` respected (scroll-reveal already complies —
      keep it true)

## hygiene

- [ ] browser console clean (no errors; investigate new warnings)
- [ ] before/after screenshots captured for the work item's verification notes
- [ ] no dev-gate weakening (exposure boundary flags untouched)

## what it is not

Not a substitute for the test suite ([[testing-strategy]]) or code review — it runs alongside
them at stage 6–7 of [[implementation-lifecycle]].
