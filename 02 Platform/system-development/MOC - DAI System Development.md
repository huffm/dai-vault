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
- [[WI-0009-gamepk-propagation-through-competition-matchup-input]] — Propagate gamePk Through
  CompetitionMatchupInput v1: optional exact event identity on the initiating generation
  request so doubleheaders become safely capturable instead of only safely rejected; completes
  the WI-0006 seam (**complete, local only** 2026-07-13; authorized via the WI-0008 planner
  decision gate 12/12; 1127/1127 tests incl. 7 new + live non-paid regression; **complete +
  integrated** 2026-07-13, dai `d493f84` on main and pushed, origin synced; branch pushed +
  retained)
- [[WI-0010-planner-evidence-fidelity]] — Planner Evidence Fidelity v1.1: MOC integration
  fallback (WI-0001 false-unknown), delivered-candidate exclusion + explicit-alias dedup,
  structured reconciliation planning readout; four defects reproduced by the first
  post-WI-0009 planning run (**complete + integrated** 2026-07-14, dai `e64567f` on main and
  pushed, origin synced; 73/73 fixture asserts re-verified on main 2026-07-14; strict
  real-state run zero warnings; branch deleted local + remote — deviation from the
  branch-retained convention of WI-0004..0009)
- [[WI-0011-buyer-decision-brief-contract]] — Buyer Decision Brief Contract v1: canonical
  server-owned buyer pregame brief (GET /brief + deterministic markdown export at
  /brief/markdown); numeric confidence and threshold labels removed from every buyer
  surface (panel, buyer artifact wire, history, landing); persisted identity incl. gamePk;
  evidence-gated band as sole strength language; deterministic market-context copy via the
  fidelity guard; first V1 critical-path item (**complete + integrated** 2026-07-14, dai
  `140b5a2` fast-forwarded to main and pushed, origin synced; 1176/1176 C# + 134/134
  vitest re-verified pre-integration; review 10 findings fixed; live-verified on
  823845/822882; branch pushed + retained)
- [[WI-0012-settled-outcome-recap]] — Settled Outcome Recap v1: canonical buyer postgame
  recap (GET /recap + deterministic markdown at /recap/markdown) embedding the WI-0011
  brief verbatim; closed fail-closed state vocabulary incl. honest `no_result` for
  non-final settlements; persisted evaluation is the sole correctness source; no-position
  never scored; excluded renders exact non-evaluation copy; per-read only, no aggregates;
  all four buyer surfaces consolidated onto one loader + suppression tripwire; second V1
  critical-path item (**complete + integrated** 2026-07-14, dai `7152818`
  fast-forwarded to main and pushed, origin synced -- pure ff, tree-identical;
  1212/1212 C# + 134/134 vitest re-verified pre-integration; review 9 fixed / 1
  refuted; live-verified on 823845/823357/no-position/unsettled; branch pushed +
  retained)
- [[WI-0013-pilot-ops-hardening]] — Pilot Operations Hardening v1: fail-closed 409
  duplicate active-run creation guard (identity precedence gamePk-equality over
  orientation-insensitive matchup; per-identity in-process creation gate; excluded/failed
  never block; distinct doubleheader gamePks independently creatable; tenant-scoped,
  zero-spend/zero-row rejection); metering price coverage for every configurable model
  (gpt-4.1-mini added) with explicit pricingStatus + loud unpriced warning; V1 concierge
  operations runbook + delivery-ledger/operator-time templates + RC drill package
  (prepared, NOT executed; four separate authorization gates); third and final V1
  critical-path item (**complete + integrated** 2026-07-14, dai `85a8831`
  fast-forwarded to main and pushed, origin synced -- pure ff, tree-identical;
  1235/1235 C# + 453/453 pytest + 134/134 vitest re-verified pre-integration; review
  12 fixed / 2 refuted; branch pushed + retained; RC drill NOT executed, separately
  gated; single-operator-host constraint standing)
- [[WI-0020-platform-hardening-catalog-corrections]] — AI Engineering Hardening
  Catalog v1.1 Corrections: canonical seven-class evidence taxonomy applied across the
  four hardening docs; one branch policy (every pulled queue card branches, vault-docs
  included, no direct-to-main, ff-only separately authorized integration); G-10 recast
  as credential exposure classification + preparation with R-05-only remediation and
  the corrected exposure facts (Development.json untracked/never committed; sa
  password historically exposed in tracked appsettings.json until `ded9969`); fitness
  count fixed at fourteen adopted = eleven existing + three added with FC-C1..C3
  conditional and excluded (**complete + integrated** 2026-07-15, docs-only, dai-vault
  `5c2500b` fast-forwarded to main -- pure ff, tree-identical; branch pushed +
  retained; zero spend, zero writes, zero credential access; G-10 unexecuted, R-05
  gated, G-01 unresolved)
- [[WI-0021-protocol-hardening-candidate-specifications]] — AI Engineering Protocol
  Hardening Candidate Specifications v1: exactly six branch-ready implementation
  specifications (PH-01 failure corpus, PH-02 trace completeness, PH-03 abstention
  invariants, PH-04 artifact contract invariants, PH-05 delivery/entitlement guard,
  PH-06 tool authorization fitness) with 40 fields each, falsifiable acceptance
  criteria, Inspect/Prove/Guard packaging, WI-0020 evidence taxonomy, RC-neutral vs
  RC-affecting policy, readiness verdicts (3 READY / 2 open-question / PH-05 NOT
  READY); WIP-policy correction 361a2e6 (pulled PH-06 counts as active implementation
  branch) (**complete + integrated** 2026-07-15, docs-only, dai-vault e3a5eb3+361a2e6
  fast-forwarded to main -- pure ff, tree-identical; branch pushed + retained; zero
  spend, zero writes; NO candidate activated or pulled)
- [[WI-0022-representative-protocol-failure-corpus]] — Representative Protocol
  Failure Corpus v1 (PH-01 Green subset, first hardening pull): deterministic
  15-class failure corpus (schema 1.0) + integrity harness + real-seam
  characterization tests (+27 C# / +3 python; suites 1262/456 green);
  review-corrected classifications 9 guard_verified / 3 behavior_characterized /
  1 guard_missing / 1 policy_blocked / 1 not_applicable, every gap owned by
  PH-02..PH-06; evidence class contract-represented + fixture-proven only
  (**complete + reviewed + integrated** 2026-07-15: dai 85a8831 -> `a0ca54d`
  tests-only ff with RC-equivalence record + runtime smoke; vault `b7c6842`;
  branches pushed + retained; PH-01 CLOSED; WIP freed; zero spend/writes)

- [[WI-0023-tool-authorization-fitness]] — Tool Authorization Fitness v1 (PH-06 Green
  subset, second hardening pull): deterministic 41-capability inventory (10 registered
  tools + 8 provider integrations + 16 externally effective routes + 7 operational
  procedures) + declaration contract + static drift/invalid-combination harness +
  real-seam tests (+17 C# / +3 python; suites 1279/459 green); enforcement 6 enforced /
  8 partially_enforced / 8 procedural / 2 ABSENT / 3 n/a; the two ABSENT findings
  (anonymous paid reference route + unauthenticated paid model service) dispositioned
  **conditional_rc_risk** — loopback-safe in V1, Gate-1 bind verification required;
  tests-and-declarations only, no runtime enforcement added (**complete + integrated**
  2026-07-15: dai a0ca54d -> `3f244c8` tests-only ff with RC-equivalence + risk-
  disposition records; vault `6036897`; branches pushed + retained; PH-06 Green subset
  CLOSED, Amber unpulled; WIP freed; zero spend/writes)

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
