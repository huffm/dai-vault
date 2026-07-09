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
(reduce `padding-right` by the letter-spacing, or set `letter-spacing` compensation) or the
text sits visibly off-center in an oval.

**Rationale.** The dev-artifact-review page shipped three chip dialects (hero badges,
ad-hoc header chips, `.artifact-status`); the misaligned "v2 era" chip was an instance of the
missing primitive, not a one-off CSS bug.

**Code reference.** `.artifact-status` in
`dai/apps/sports-app/src/app/dev-artifact-review/dev-artifact-review.component.scss` is the
closest existing implementation and the seed for the primitive.

**Known violations (2026-07-09).** Regime-era chip and hero badges built inline in
`dev-artifact-review.component.html` (no `line-height: 1`, no min-height); to be resolved by
[[WI-0001-chip-primitive-and-long-token-treatment]].

## R2 — long-token treatment

**Statement.** Identifiers (route keys, recipe ids, hashes, GUIDs) never use `break-all`.
Long tokens either (a) wrap only at separator boundaries, or (b) middle-truncate with the
full value available via title attribute and a copy control. Every truncated token gets a
copy affordance.

**Rationale.** `break-all font-mono` shatters identifiers mid-token, which destroys
scanability exactly where precision matters most (calibration triage).

**Code reference.** The assembled-hash row in `dev-artifact-review.component.html`
(copy-hash button) already implements the copy affordance correctly — generalize it.

**Known violations (2026-07-09).** Route key, recipe id, observed-regime fields in the Run
Anatomy section use `break-all`. Resolved by WI-0001.

## R3 — semantic status tokens

**Statement.** The four status tones (info / success / warning / error) are defined once as
CSS custom properties in `dai/apps/sports-app/src/styles.css` and referenced everywhere a
status color appears. Color semantics: green = success/settled/available only; amber =
warnings only; red = true errors only; blue = in-flight only.

**Rationale.** The tones are currently duplicated as rgba literals in three places
(`status-banner.ts` styles, `.artifact-status--*` in component SCSS, queue-row active
styles) — consistent today, guaranteed drift tomorrow.

**Known violations (2026-07-09).** The three copies above. Resolved by WI-0001.

## adding a rule

New rules arrive through the promotion gate ([[operating-model]]): observed on a real
surface, recorded in a work item's lessons, promoted here with references and violations.

## recommended next slice

Execute WI-0001, then update the "known violations" entries to resolved-with-commit.
