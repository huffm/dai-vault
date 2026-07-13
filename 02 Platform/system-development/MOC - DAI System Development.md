---
title: "MOC - DAI System Development"
type: "moc"
date: "2026-07-09"
status: "in-progress"
project: "DAI"
slice: "DAI System Development v1"
repos:
  dai: "docs-only"
  dai-vault: "docs-only"
tags:
  - system-development
  - moc
  - okf
related:
  - "02 Platform/system-development/operating-model.md"
  - "06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md"
---

# MOC - DAI System Development

Hub for the DAI development operating model: how work is specified, traced, implemented,
tested, and turned into reusable knowledge. Execution mechanics stay in the slice doctrine
(`06 Execution/agent-slice-workflow-doctrine-v1.md`); this subtree governs intent and
traceability around it.

## front matter placement decision (recorded here on purpose)

The canonical 9-field OKF schema (`06 Execution/patterns/okf-yaml-front-matter-pattern-v1.md`)
is documented for `06 Execution` docs. This subtree deliberately extends the same schema —
same nine fields, same allowed values — to `02 Platform/system-development/`. One new `type`
value is introduced: `moc` (no existing value fits a map-of-content). This is a placement
extension, not a second dialect. A future hygiene audit should read this as intent, not drift.

## operating model

- [[operating-model]] — the spine (work item / spec / code), meaningful-change threshold, six lenses, learning loop
- [[work-item-traceability]] — IDs, naming, link topology, local-spine mode until Azure DevOps is wired
- [[implementation-lifecycle]] — intake → spec → gate → branch → implement → verify → review → close

## implementation doctrine

- [[frontend-implementation]] — Angular/sports-app repo patterns (seeded, in-progress)
- [[backend-implementation]] — platform API repo patterns (seeded, in-progress)
- [[architecture-contracts]] — truth hierarchy, buyer boundary, correlation, error semantics
- [[testing-strategy]] — what must be tested per change class

## design system

- [[component-rules]] — chip primitive, long-token treatment, status color tokens
- [[interaction-states]] — the shipped status/feedback system (StatusDescriptor, queue states)
- [[visual-qa-checklist]] — pre-completion visual gate

## work items

- [[_template]] — tiered work-item spec template (lite core + feature-class sections)
- [[WI-0001-chip-primitive-and-long-token-treatment]] — chip primitive, long-token treatment,
  status tokens (**complete** 2026-07-09; first proving run; integrated to dai main
  `bb10c3c` and pushed, branch deleted)
- [[WI-0002-artifact-chip-primitive-alignment]] — evaluate aligning the squared
  `.artifact-chip` tags with the chip system (**BACKLOG, not authorized**; disposition
  open, activation gate in spec)
- [[WI-0003-shared-chip-and-long-token-module-promotion]] — promote `.chip` +
  `app-long-token` to a shared boundary (**BACKLOG, not authorized**; gated on a concrete
  second consumer)
- [[WI-0004-platform-api-shutdown-process-match]] — Truthful Platform API Shutdown v1:
  `stop-platform-api.ps1` now verifies port release + process exit before reporting success,
  targets the real listener owner, and refuses to kill an unrelated port owner (**complete +
  integrated** 2026-07-11; dai `e8050a9` fast-forwarded to main and pushed, origin synced;
  unit 15/15 + live scenarios A-D; branch pushed + retained)
- [[WI-0005-starter-retrieval-caches-transport-failures]] — Identity-Safe Starter Cache v1:
  `MlbStarterClient` no longer caches transport failures as "no starters announced"; cache admits
  only fully-grounded results, identity-safe key, configurable TTL, cancellation propagated
  (**complete + integrated** 2026-07-11; dai `4693b9d` fast-forwarded to main and pushed,
  origin synced; 1092/1092 tests; branch pushed + retained)
- [[WI-0006-identity-safe-mlb-doubleheader-resolution]] — Identity-Safe MLB Doubleheader
  Resolution v1: a matchup is not an event identity. `/source-readiness` takes an optional
  `gamePk`; an ambiguous same-day matchup now fails closed with explicit candidates instead of
  `FirstOrDefault`-ing whichever game the provider listed first; starter cache re-keyed to
  `starters:v2:statsapi:mlb:{gamePk}` so game 1 and game 2 cannot collide (**complete +
  integrated** 2026-07-13; dai `4f8f381` fast-forwarded to main and pushed, origin synced;
  1120/1120 tests; live-verified on the real 2026-07-11 MIL@PIT split DH incl. adversarial
  cache ordering both directions; branch pushed + retained)
- [[WI-0007-mandatory-work-items-and-slice-synopsis-workflow]] — Mandatory Structured Work
  Items and Slice Synopsis Workflow v1: qualification gate owned by `dai-slice-runner`
  (qualifying slices need a WI before execution), canonical Slice Synopsis owned by
  `dai-agent-handoff` as every handoff's mandatory final section (**complete + integrated**
  2026-07-13; dai `41e0a46` fast-forwarded to main and pushed, origin synced; validation =
  single-owner greps + 6 desk-check scenarios; branch pushed + retained)
- [[WI-0008-evidence-grounded-next-slice-planner]] — Evidence-Grounded Next-Slice Planner v1:
  read-only deterministic planning snapshot (`scripts/dev/planning/`) + `dai-next-slice-planner`
  skill + operator-owned [[platform-delivery-timeline-v1]]; planner recommends one bounded next
  slice with provenance, never creates/authorizes/executes (**complete + integrated** 2026-07-13; dai `88c9f09` fast-forwarded to main and pushed, origin synced; 45/45 fixture asserts + real-state run + planner desk-run; branch pushed + retained)

## scope boundary of this registry

`WI-####` items cover qualifying slices per the WI-0007 qualification rule (canonical
categories in the `dai-system-development` skill): implementation, investigation/discovery,
integration and closeout, operational **remediation**, migrations, workflow-governance,
doctrine or skill changes, measurement/validation, and documentation that changes canonical
operating behavior. A slice does not stop qualifying because it is small, docs-only,
script-only, single-session, or ends blocked/discovery-only.

Routine governed operational **cadence** (capture, settlement, calibration readouts executed
under an existing OKF plan) remains outside this taxonomy per [[operating-model]] and is
governed by OKF records under `06 Execution/`. Precedent preserved:
`06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md` is a governed
operational slice that deliberately did **not** mint a WI, while the code defect it
discovered did (WI-0004). Genuinely clerical edits (typo, broken link, formatting-only,
hash update in an open WI) proceed without a WI but record the explicit non-qualifying
reason (WI-0007).

## skill layer

- `dai/.claude/skills/dai-system-development/SKILL.md` — the workflow router (coordinates
  existing skills; never replaces them). Tracked in
  `06 Execution/skills/dai-skills-inventory-v1.md`.
