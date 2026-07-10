---
title: "WI-0001 Chip primitive and long-token treatment"
type: "plan"
date: "2026-07-09"
status: "complete"
project: "DAI"
slice: "WI-0001 chip primitive and long-token treatment"
repos:
  dai: "code+docs"
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

**COMPLETE — approved 2026-07-09, executed through all 8 lifecycle stages on branch
`wi/0001-chip-primitive`. First proving run of the system-development model.**

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

## implementation summary

One `.chip` primitive in `styles.css` (sizes `--badge`/`--status`; tones
`--muted/--info/--success/--warning/--error`; modifiers `--ghost`/`--quiet`) replaced four
chip dialects. Uppercase status chips carry trailing letter-space compensation
(`padding-right: calc(0.6rem - 0.08em)`), which is what fixed the "v2 era" optical
misalignment — a primitive property, not a per-chip patch. Long identifiers render through
a new `app-long-token` component: `splitToken()` (pure, TDD) cuts after each separator and
the template joins segments with `<wbr>`, so wrapping happens only at boundaries;
`overflow-wrap: anywhere` is the emergency fallback for separator-less segments such as a
bare sha256. Run id and assembled hash gained copy affordances. Four status tones are now
`--status-*` custom properties consumed by chips, the status banner, queue-row selection,
and the signal-coverage tags.

## files changed

- `apps/sports-app/src/styles.css` — status tokens + chip primitive (+89)
- `apps/sports-app/src/app/dev-artifact-review/long-token.ts` — new component + `splitToken`
- `apps/sports-app/src/app/dev-artifact-review/long-token.spec.ts` — new, 5 tests
- `apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.html` — all chips migrated; 5 long-token sites
- `apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.{ts,scss}` — chip tone maps; local chip styles removed; tokens consumed
- `apps/sports-app/src/app/dev-artifact-review/status-banner.ts` — tones from tokens
- `apps/sports-app/src/app/dev-artifact-review/run-anatomy.{ts,spec.ts}` — `chip--*` vocabulary; new `recipeLabel()`

9 files, +344/-146. No backend, contract, DTO, doctrine, or buyer-surface change.

## tests run

`npx ng test --watch=false` → **139 passed** (was 130; +5 `splitToken`, +4 `recipeLabel`;
`guardBadgeClass` specs updated to the chip tone vocabulary). `npx ng build` → clean.

## verification results

- `grep -rn "break-all" app/dev-artifact-review/` → 1 hit, the rule comment in `long-token.ts`. PASS
- `grep -rn "artifact-status" app/` → 0 hits. PASS
- status rgba literals → success/warning/error: one definition site each, in `styles.css`.
  PASS. **info (`77,141,255`): PARTIAL** — one status-token definition, plus 7 pre-existing
  accent uses, because `--app-accent: #4d8dff` is the same rgb. The original acceptance
  criterion ("appear exactly once") was met for three of four tones; the fourth was verified
  in the final review with a grep that had been wrongly scoped to exclude `styles.css`. Not a
  defect (accent ≠ status), but the earlier PASS was overstated. See [[component-rules]] R3.
- Computed-style proof of the review fix: `.chip--quiet` resolves to `#92a6c1`
  (`--app-text-muted`), plain `.chip--badge` to `#b8c5d8`. PASS
- Table status chips contain no underscores (snake_case routed through `formatLabel`). PASS

## visual QA result

Per [[visual-qa-checklist]], real rendered app against the live API:

- **1440 px** — no overflow; "v2 era" chip optically centered; guard chip aligned; route key
  and observed regime wrap at separators. PASS
- **200 % zoom** — chip centering holds. PASS (acceptance criterion 2)
- **768 px** — no overflow. PASS
- **390 px** — no overflow. Initially FAILED (`scrollWidth` 539 > 375, caused by the bare
  sha256 having no separators); fixed by the `overflow-wrap: anywhere` fallback and
  re-verified. This is exactly the defect class the checklist exists to catch.
- Console: 0 errors, 0 warnings. Copy affordances present on run id + hash.

## docs updated

- [[component-rules]] — R1/R2/R3 known violations marked resolved with commit `255e4ae`;
  R1 gained the layered-utility rule learned during review.
- [[frontend-implementation]] — promoted patterns 7 (chip primitive / token location) and
  8 (unlayered component classes vs Tailwind utilities).

## links

- work item: WI-0001 (ADO: not wired — local-spine mode, per user decision)
- branch: `wi/0001-chip-primitive`
- pr: none — branch retained locally, awaiting user decision on merge
- commits: `255e4ae` (primitive + tokens + long-token), `d20279b` (390 px overflow fix),
  `49dbea3` (code-review fixes), `bb10c3c` (final review: chip modifier/tone cascade order)
- tests: `long-token.spec.ts` (new), `run-anatomy.spec.ts` (updated) — 139 green
- verification notes: this section + slice handoff in `06 Execution/handoffs/current-slice.md`
- docs updated: `component-rules.md`, `frontend-implementation.md` (above)
- lessons: 3 promoted (below); 1 deferred

## remaining risks

- ~~**Partial review coverage.**~~ **Closed 2026-07-09.** The two lenses that died on an API
  session limit (line-by-line scan; visual/a11y + responsive) were re-run to completion in a
  dedicated review pass. Findings: 1 minor cascade-order defect (fixed, `bb10c3c`), 2
  documentation inaccuracies (fixed), 1 false positive (regex lookbehind — the project's
  resolved browser targets are Safari 26.x / iOS 18.5+, which support it). No blocker or
  significant defect found.
- `.chip--badge` sets `color`, so any Tailwind `text-*` utility on a chip is silently
  defeated. Mitigated by `chip--quiet` and documented in R1, but the trap remains for the
  next author who reaches for a utility.
- Chip geometry is verified by computed style and manual measurement, not by an automated
  test. A future CSS change could regress optical centering without failing the suite. The
  project deliberately avoids TestBed/DOM tests; visual QA is the standing control.

## follow-up items

1. **WI-0002 (candidate):** fold `.artifact-chip` (squared signal-coverage tags) into the
   chip primitive as a size variant. It survived this item as a genuinely distinct shape;
   the SCSS comment now says so honestly rather than overclaiming.
2. **WI-0003 (candidate):** promote `.chip` and `app-long-token` out of the dev page into a
   shared UI module once a second surface needs them (currently one consumer — not yet
   doctrine).
3. ~~Run a full `/code-review` on the branch to cover the two lenses that died.~~ Done
   2026-07-09; see remaining risks. Branch is merge-ready with minor notes.
4. **Candidate cleanup (not a defect):** `--status-info-rgb` restates the rgb of
   `--app-accent`. A future item could derive one from the other.

## reusable lessons (promoted)

1. A misaligned pill is a missing primitive, not a CSS bug. The fix that mattered was the
   trailing letter-space compensation on the *primitive*, which corrected every uppercase
   chip at once.
2. Tailwind v4 utilities live in `@layer utilities`; unlayered component classes beat them
   regardless of source order or specificity. Component classes that set `color` must expose
   modifiers rather than expecting utilities to override.
3. Separator-aware wrapping needs an emergency fallback: a token with no separators (a bare
   hash) will otherwise force page-level horizontal overflow at narrow widths — invisible at
   1440 px, fatal at 390 px.
4. A grep proof is only as good as its scope. The "each status literal appears once" claim
   passed because the grep excluded the file that holds the counter-examples. Scope every
   verification grep to the whole search space, or state the scope in the claim.
5. In a flat single-class system (BEM-ish modifiers), source order *is* the API. Modifiers
   must be declared after the classes they are meant to override, and combinations must be
   checked by computed style — reading the stylesheet top-to-bottom hides the bug.

## final handoff requirements

Met: definition of done in [[implementation-lifecycle]] checked (8/8 links, docs actioned,
lessons recorded); slice handoff appended to `06 Execution/handoffs/current-slice.md`.
