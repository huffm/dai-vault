---
title: "WI-0007 Mandatory Structured Work Items and Slice Synopsis Workflow v1"
type: "plan"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0007 Mandatory Structured Work Items and Slice Synopsis Workflow v1"
repos:
  dai: "code (skill layer only)"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - workflow
  - governance
  - skills
related:
  - "02 Platform/system-development/operating-model.md"
  - "02 Platform/system-development/work-item-traceability.md"
  - "02 Platform/system-development/implementation-lifecycle.md"
  - "06 Execution/skills/dai-skills-inventory-v1.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# WI-0007 mandatory structured work items and slice synopsis workflow v1

**Slice type:** workflow-governance (doctrine + skill layer). **Opened:** 2026-07-13.

## problem

Two workflow invariants exist only as prompt-by-prompt instructions and one session memory,
not as canonical doctrine the skill layer enforces:

1. qualifying slices must be represented as structured OKF work items **before** execution --
   today `dai-system-development` has a "meaningful-change threshold" but nothing forces the
   qualification question to be asked and answered at slice start, and the MOC's scope boundary
   still reads as "WI-#### covers code work" only;
2. every completed slice handoff must end with a concise, fixed-format **Slice Synopsis** --
   currently requested per-prompt, not owned by any skill.

Without enforcement, slices can mutate production, scripts, workflow, or doctrine with no
durable spine record, and handoffs end without a skimmable bottom line.

## why now

WI-0006 ran correctly but only because its prompt spelled the discipline out. The operator has
now issued the canonical contract (qualification gate + synopsis format); it belongs in the
skill layer, once, rather than restated in every prompt.

## governing objective / intended outcome

Any qualifying slice cannot begin production, script, workflow, skill, or doctrine changes
until its structured work item exists (or the governing WI is explicitly identified for an
integration continuation). Every execution/implementation/integration/closeout handoff ends
with the mandatory `Slice Synopsis` section as its final section. Both rules live in exactly
one canonical place each, enforced by the skills that own those seams.

## scope

- `dai/.claude/skills/dai-slice-runner/SKILL.md` -- owns the qualification gate; SLICE START
  gains qualification fields; SLICE CLOSE gains work-item disposition + mandatory Slice Synopsis.
- `dai/.claude/skills/dai-agent-handoff/SKILL.md` -- handoffs must identify governing WI,
  status, OKF path, MOC disposition, and end with the Slice Synopsis (canonical format defined
  here, referenced elsewhere).
- `dai/.claude/skills/dai-skill-router/SKILL.md` -- routing rule: surface the qualification
  requirement for any likely slice.
- `dai/.claude/skills/dai-docs-architect/SKILL.md` -- owns OKF placement/front matter/MOC
  registration for work items; born-correct rule restated once.
- `dai/.claude/skills/dai-system-development/SKILL.md` -- qualification categories, status
  discipline (implementation-complete vs merge-ready vs integrated vs closed), integration
  continuation rule, and the meta-work rule (workflow/skill/doctrine changes follow the same
  discipline).
- `dai-vault`: this WI; MOC registration + scope-boundary reconciliation; skills inventory
  update (same-slice rule per `dai-write-skill`); current-slice append; handoff.

## out of scope

- runtime/application code, tests, prompts, registry, confidence, decision, market,
  reconciliation, calibration, buyer surfaces, schema, Angular;
- Azure DevOps wiring; new OKF categories; a new competing WI schema (the existing
  `_template.md` remains the spec);
- rewriting historical WIs or moving existing artifacts (born-correct rule: existing artifacts
  move only when materially touched or a concrete retrieval problem requires it);
- WI-0002/0003; any pending sports work.

## architecture / ownership boundary

One source of truth per rule: the **qualification gate** is owned by `dai-slice-runner` (the
lifecycle owner); the **Slice Synopsis format** is defined once in `dai-agent-handoff` (the
output-contract owner) and referenced -- not restated -- by `dai-slice-runner`. Qualification
*categories* live in `dai-system-development` (the intake owner). Other skills point at these,
never duplicate them.

## key decisions

1. **Canonical statuses stay as found in the repo.** Front matter uses the established
   lowercase set (`in-progress`, `complete`, `blocked`, `superseded` per `_template.md`);
   the finer dispositions (implementation complete / merge ready / integrated / closed) are
   body-level facts recorded in the links block and MOC entry, exactly as WI-0005/0006 already
   practiced. The uppercase lifecycle list from the operator prompt is mapped, not imported.
2. **MOC scope boundary reconciled, not reversed.** Routine governed operational *cadence*
   (capture/settlement under an existing OKF plan) still does not mint a WI -- the
   v2-settlement precedent stands. Operational *remediation*, investigation/discovery,
   migration, measurement/validation, and workflow/doctrine changes now qualify. The MOC
   boundary text is updated to say this and cites WI-0007.
3. **Integration continuation** continues the governing WI (as WI-0006 did) rather than
   minting a duplicate; a new WI appears only when integration uncovers a distinct engineering
   problem.
4. **Non-qualifying work requires an explicit recorded reason** ("too small" alone is
   insufficient); the two-line `Work-item qualification: does not qualify / Reason: ...` form
   is the record.

## risks

- doctrine duplication across five skills drifting apart -> mitigated by single-owner rule
  above; each skill references the owner instead of restating;
- ceremony creep on genuinely clerical edits -> mitigated by the explicit non-qualifying path;
- contradiction with the MOC's existing scope boundary -> reconciled in the same slice (one
  source of truth);
- skill files are prompts, not code -- no test suite can enforce them -> mitigated by the six
  desk-check validation scenarios below recorded with expected outcomes.

## guardrails

Local commits only; nothing pushed. No runtime code. Skill edits minimal and additive.
`dai-write-skill` same-slice rule honored (skills inventory updated with the skill changes).

## acceptance criteria

1. `dai-slice-runner` requires the four-line qualification gate before any qualifying change
   and blocks qualifying execution until the WI exists or is identified.
2. `dai-agent-handoff` defines the canonical Slice Synopsis (Change/Reason/Proof/State/Next,
   ~100 words, final section) and requires governing-WI/status/OKF-path/MOC-disposition fields.
3. `dai-skill-router` routes the qualification requirement at slice start.
4. `dai-system-development` enumerates qualifying categories, the status-discipline mapping,
   the integration-continuation rule, and the meta-work rule.
5. `dai-docs-architect` owns WI OKF placement/front matter/MOC registration.
6. The MOC scope boundary reflects the broadened qualification rule without invalidating the
   routine-cadence precedent.
7. Skills inventory updated in the same slice.
8. All six validation scenarios desk-check to their expected outcomes.
9. Focused local commits exist in both repos; nothing pushed.

## verification plan

No executable test surface (markdown skill layer). Verification = (a) grep-level confirmation
that each rule exists in exactly one owning skill and is referenced elsewhere; (b) desk-check
of the six operator-specified scenarios (new implementation, discovery-only, integration
continuation, workflow/doctrine change, clerical correction, exact-section-list prompt)
against the updated skill texts; (c) `git diff` confined to the scoped files.

## rollback / reversibility

Two focused commits (dai skills; vault docs). Revert both and the workflow returns exactly to
the pre-WI-0007 state; no runtime, schema, or persisted-data impact.

## dependencies / prerequisites

None beyond the existing skill layer and OKF doctrine. Azure DevOps intentionally not wired
(local-spine mode per `work-item-traceability.md`).

## approval boundary

WI-0007 authorizes skill-layer and vault-doctrine changes for these two invariants only. It
does not authorize paid calls, sports work, runtime changes, pushes, merges, PRs, or the
creation of other work items.

## deferred work

- localization of Jera-inherited skill bodies (pre-existing inventory note);
- `dai-vault-doc-conventions` low-level formatting skill (pre-existing candidate);
- Azure DevOps wiring for the WI spine.

## validation record (2026-07-13)

Single-owner proof by grep: the synopsis format block exists only in `dai-agent-handoff`;
the qualifying-categories enumeration only in `dai-system-development`; the blocking
four-line gate only in `dai-slice-runner`; the other skills reference by name. All six
operator-specified scenarios desk-checked against the updated texts and passed: new
implementation (gate -> WI before code), discovery-only (qualifies, blocked/discovery-only
still qualifies), integration continuation (continues governing WI, no duplicate -- matches
the WI-0006 integration precedent), workflow/doctrine change (meta-work rule; WI-0007 itself
is the live proof), clerical correction (non-qualifying with recorded specific reason), and
exact-section-list prompt (requested sections preserved, synopsis appended last). MOC scope
boundary reconciled: operational cadence precedent preserved, remediation/investigation/
migration/measurement/workflow now qualify.

## links

- work item: WI-0007 (ADO: AB#-- when wired)
- branch: `wi/0007-workflow-governance` (dai, from `4f8f381`) -- local only, not pushed
- pr: -- (not authorized)
- commits: dai (5 SKILL.md files) + dai-vault (this WI, MOC, skills inventory, current-slice,
  handoff) -- hashes recorded in the slice handoff at close
- tests: none (no executable surface); validation = single-owner greps + 6 desk-check scenarios
- verification notes: see validation record above; `git diff` confined to the scoped files
- docs updated: this WI; `MOC - DAI System Development.md` (registration + scope boundary);
  `06 Execution/skills/dai-skills-inventory-v1.md` (same-slice rule); current-slice append;
  slice handoff
- lessons: workflow invariants restated per-prompt eventually drift or get skipped -- the
  durable home for a lifecycle rule is the skill that owns that seam, with exactly one owner
  per rule and references elsewhere

## final disposition

Implementation complete, local only (2026-07-13). Integration and push separately gated.
