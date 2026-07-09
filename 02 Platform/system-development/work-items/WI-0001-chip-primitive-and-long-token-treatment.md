---
title: "WI-0001 Chip primitive and long-token treatment"
type: "plan"
date: "2026-07-09"
status: "in-progress"
project: "DAI"
slice: "WI-0001 chip primitive and long-token treatment"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - design-system
  - ui
related:
  - "02 Platform/system-development/design-system/component-rules.md"
  - "02 Platform/system-development/work-items/_template.md"
---

# WI-0001 chip primitive and long-token treatment

**DRAFT — awaiting review before any app change. This spec is the proving run of the
system-development model.**

## problem

Three verified component-rule violations on `/dev/artifacts` (dev-artifact-review):

1. Three chip dialects on one page — hero badges and the regime-era chip are built inline in
   `dev-artifact-review.component.html` without `line-height: 1`/min-height (the visibly
   misaligned "v2 era" pill), while `.artifact-status` is a proper class. Violates
   [[component-rules]] R1.
2. Identifier fields (route key, recipe id, observed regime) use `break-all font-mono`,
   shattering tokens mid-identifier. Violates R2.
3. Status tone rgba values are duplicated in three places (`status-banner.ts` styles,
   `.artifact-status--*` SCSS, queue-row active styles). Violates R3.

## desired behavior

One chip primitive with size (badge/chip/status) and tone (muted/info/success/warning/error)
variants used by every pill on the page; "v2 era" renders optically centered. Long
identifiers wrap at separator boundaries or middle-truncate with a copy affordance. Status
tones defined once as custom properties in `styles.css` and referenced everywhere.

## affected surfaces

- `dai/apps/sports-app/src/styles.css` (status tone tokens; possibly chip primitive if shared
  beyond the page)
- `dai/apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.scss` + `.html`
- `dai/apps/sports-app/src/app/dev-artifact-review/status-banner.ts` (consume tokens)

## non-goals

- No page-hierarchy/elevation redesign (separate item).
- No changes to decision/prompt/calibration/reconciliation/confidence/artifact doctrine,
  buyer surfaces, routes, or exposure gates.
- No table consolidation (open question in the operating model).

## acceptance criteria

1. Every pill on `/dev/artifacts` uses the chip primitive; zero inline ad-hoc chip styling
   remains in the template.
2. The regime-era chip text is optically centered at 100% and 200% zoom.
3. No identifier on the page renders with `break-all`; truncated identifiers expose the full
   value (title attr) and a copy control.
4. Grep proof: the status rgba literals appear exactly once in the app source (as token
   definitions).
5. Frontend suite green; visual QA checklist passes at 390/768/1440.

## test plan

- Extend `dev-artifact-review` pure-module specs only where logic changes (copy affordance
  helper, if extracted). Chip/token work is CSS-structural: covered by acceptance greps +
  visual QA rather than DOM tests (per [[testing-strategy]] — TestBed avoided).
- Existing 130-test suite must stay green.

## implementation notes

Seed the primitive from `.artifact-status`; add size/tone modifiers; migrate hero badges and
regime chip. Introduce `--status-info/success/warning/error` custom properties in
`styles.css`; consume from banner styles, chip classes, queue-row. Long-token: prefer a small
`.long-token` class + reusable copy-button pattern generalized from the assembled-hash row.
No backend or contract changes.

## docs to update

- [[component-rules]]: mark R1/R2/R3 known violations resolved with commit hash.
- [[frontend-implementation]]: promote the token/primitive locations as pattern entries.

## verification commands

- `cd dai/apps/sports-app && npx ng test --watch=false`
- `npx ng build`
- `grep -rn "break-all" src/app/dev-artifact-review/` → expected: no identifier fields
- grep for the four status rgba literals → expected: one definition site
- dev server + [[visual-qa-checklist]] pass (390/768/1440, all states, chip alignment)

## risks

- Chip migration touches many template spans — visual regression risk; mitigate with
  before/after screenshots per section.
- Token extraction could shift perceived colors if any of the three copies had drifted —
  diff the three value sets first; if they differ, that drift is itself evidence for R3.

## links

- work item: WI-0001 (ADO: not wired yet — local-spine mode)
- branch: — (proposed: `wi/0001-chip-primitive`)
- pr: —
- commits: —
- tests: —
- verification notes: —
- docs updated: —
- lessons: —

## final handoff requirements

Per [[implementation-lifecycle]] definition of done; slice handoff appended to
`06 Execution/handoffs/current-slice.md`.
