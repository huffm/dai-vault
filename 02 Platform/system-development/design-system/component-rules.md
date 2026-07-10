---
title: "DAI Component Rules"
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
  - components
  - ui
related:
  - "02 Platform/system-development/design-system/interaction-states.md"
  - "02 Platform/system-development/work-items/WI-0001-chip-primitive-and-long-token-treatment.md"
---

# dai component rules

## purpose

Testable rules for reusable UI primitives. Each rule: statement → rationale → code reference
→ known violations. Values live in code; this doc cites paths and never restates rgba/rem
literals.

## R1 — chip primitive (one chip system per page)

**Statement.** Every badge/chip/status pill uses one primitive: `display: inline-flex;
align-items: center; justify-content: center; line-height: 1; white-space: nowrap`; a
consistent `min-height` and horizontal padding per size variant; no fixed width/height pairs
that force optical misalignment; consistent border radius and border treatment. Size variants
(badge / chip / status) and tone variants (muted / info / success / warning / error) — never
ad-hoc template chips.

**Optical note.** Uppercase letter-spaced text carries a trailing letter-space; compensate
(reduce `padding-right` by the letter-spacing) or the text sits visibly off-center in an oval.
Implemented as `padding: 0 calc(0.6rem - 0.08em) 0 0.6rem` on `.chip--status`.

**Layering rule (learned in WI-0001 review).** Tailwind utilities live in `@layer utilities`;
unlayered component classes beat them regardless of specificity or source order. Because
`.chip--badge` sets `color`, a `text-*` utility on a chip is silently ignored. Tone and text
color are chip modifiers (`.chip--quiet`, `.chip--muted`, tone classes) — never utilities.

**Rationale.** The dev-artifact-review page shipped four chip dialects (hero badges, ad-hoc
header chips, table pills, `.artifact-status`); the misaligned "v2 era" chip was an instance
of the missing primitive, not a one-off CSS bug. Fixing the primitive corrected every
uppercase chip at once.

**Code reference.** `.chip` and variants in `dai/apps/sports-app/src/styles.css`.

**Status: resolved 2026-07-09** by [[WI-0001-chip-primitive-and-long-token-treatment]]
(commit `255e4ae`, review fixes `49dbea3`). Remaining exception: `.artifact-chip` (squared
signal-coverage tags) is a distinct shape, not a pill; folding it in is a candidate WI-0002.

## R2 — long-token treatment

**Statement.** Identifiers (route keys, recipe ids, hashes, GUIDs) never use `break-all`.
They wrap only at separator boundaries (`_ - . : / @`), carry the full value in a `title`
attribute, and expose a copy control where the value is likely to be copied.

**Emergency-fallback rule (learned in WI-0001 visual QA).** Separator-aware wrapping alone is
insufficient: a token with no separators (a bare sha256) forces page-level horizontal
overflow at narrow widths — invisible at 1440 px, fatal at 390 px. Pair `<wbr>` segmentation
with `overflow-wrap: anywhere` so the browser prefers boundaries and breaks mid-segment only
as a last resort.

**Rationale.** `break-all font-mono` shatters identifiers mid-token, destroying scanability
exactly where precision matters most (calibration triage).

**Code reference.** `app-long-token` (`splitToken` + component) in
`dai/apps/sports-app/src/app/dev-artifact-review/long-token.ts`, with `long-token.spec.ts`.

**Status: resolved 2026-07-09** by [[WI-0001-chip-primitive-and-long-token-treatment]]
(commits `255e4ae`, `d20279b`). The component lives in the dev page until a second surface
needs it (candidate WI-0003).

## R3 — semantic status tokens

**Statement.** The four status tones (info / success / warning / error) are defined once as
CSS custom properties in `dai/apps/sports-app/src/styles.css` and referenced everywhere a
status color appears. Color semantics: green = success/settled/available only; amber =
warnings only; red = true errors only; blue = in-flight only.

**Rationale.** The tones were duplicated as rgba literals in three places (`status-banner.ts`
styles, `.artifact-status--*` in component SCSS, queue-row active styles) — consistent that
day, guaranteed drift the next.

**Code reference.** `--status-{info,success,warning,error}-rgb` and `-text` in
`dai/apps/sports-app/src/styles.css`. Consumed as `rgba(var(--status-X-rgb), a)`.

**Status: resolved 2026-07-09** by [[WI-0001-chip-primitive-and-long-token-treatment]]
(commit `255e4ae`). Grep proof: each literal appears exactly once, at its definition site.
The migration surfaced one real drift — the status banner's info title used a one-off
`#b9d2ff` — now unified to the info token.

## adding a rule

New rules arrive through the promotion gate ([[operating-model]]): observed on a real
surface, recorded in a work item's lessons, promoted here with references and violations.

## recommended next slice

Execute WI-0001, then update the "known violations" entries to resolved-with-commit.
