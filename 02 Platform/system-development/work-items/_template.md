---
title: "WI-#### <short title>"
type: "plan"
date: "YYYY-MM-DD"
status: "in-progress"
project: "DAI"
slice: "WI-#### <short title>"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
related:
  - "02 Platform/system-development/work-items/_template.md"
---

# WI-#### <short title>

<!--
tiers:
  LITE (required for every work item): problem, acceptance criteria, test plan,
    verification commands, links.
  FEATURE-CLASS (also required when the item changes behavior across surfaces, touches a
    contract, or spans multiple slices): all remaining sections.
  use `none` for a section that genuinely does not apply; do not delete section headers
  on feature-class items.
front matter: status in-progress -> complete (or blocked/superseded); repos.* reflect the
  item's actual effect at close.
-->

## problem  <!-- LITE -->

What is wrong or missing, observed where. Cite the surface/file/run, not a vibe.

## desired behavior

What the user/agent should experience after the change.

## affected surfaces

Pages, components, endpoints, docs. Paths.

## non-goals

What this item deliberately does not change (doctrine boundaries at minimum).

## acceptance criteria  <!-- LITE -->

Falsifiable statements. Each one is checkable by a command, a test, or the visual QA
checklist.

## test plan  <!-- LITE, written BEFORE implementation -->

Tests to add/update, per [[testing-strategy]] change classes. Name spec files.

## implementation notes

Approach, sequencing, any proposed contract change (which waits for review per
[[architecture-contracts]]).

## docs to update

Vault docs this item must touch at close (rule violations to mark resolved, patterns to
promote).

## verification commands  <!-- LITE -->

Exact commands run at stage 6, plus the visual QA checklist when UI is touched.

## risks

What could break; what to watch during review.

## links  <!-- LITE; all 8 required at close, per work-item-traceability -->

- work item: WI-#### (ADO: AB#— when wired)
- branch: —
- pr: —
- commits: —
- tests: —
- verification notes: —
- docs updated: —
- lessons: — (or `none`)

## final handoff requirements

Slice handoff appended to `06 Execution/handoffs/current-slice.md`; lessons promoted or
explicitly declined; definition of done in [[implementation-lifecycle]] checked.
