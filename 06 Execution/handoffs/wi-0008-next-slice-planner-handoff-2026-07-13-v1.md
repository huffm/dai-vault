---
title: "WI-0008 Evidence-Grounded Next-Slice Planner v1 -- slice handoff (2026-07-13)"
type: "handoff"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0008 Evidence-Grounded Next-Slice Planner v1"
repos:
  dai: "code (planning tooling + skill; local branch, not pushed)"
  dai-vault: "docs-only (local, not pushed)"
tags:
  - execution
  - handoff
  - system-development
  - planning
related:
  - "02 Platform/system-development/work-items/WI-0008-evidence-grounded-next-slice-planner.md"
  - "06 Execution/plans/platform-delivery-timeline-v1.md"
---

# WI-0008 slice handoff -- evidence-grounded next-slice planner v1

**Disposition: IMPLEMENTATION COMPLETE, LOCAL ONLY. Review APPROVE. Nothing pushed.**

Governing WI: WI-0008 (status `complete`, disposition implementation-complete/local; OKF path
`02 Platform/system-development/work-items/WI-0008-evidence-grounded-next-slice-planner.md`;
MOC registered).

## 1. objective and outcome

Two-layer planning system, per the governing principle "deterministic tooling assembles facts;
the planning skill evaluates tradeoffs":

- `dai/scripts/dev/planning/build-next-slice-snapshot.ps1` -- read-only, deterministic,
  versioned (schemaVersion 1.0) evidence snapshot: WI inventory with the WI-0007 status
  discipline machine-applied; deferred candidates distinct from WIs; latest-handoff selection
  by front-matter date (mtime never authority); reconciliation pointer with provenance;
  fail-closed authorization read from the operator-owned timeline block; timeline semantics
  enforced (`desired_by`/`due_by`/`not_before` distinct, `proposed_by_system` never an
  operator date, cycle/impossible-window/missing-provenance warnings); material-conflict
  warnings; every source carries precedence (1 repo/runtime .. 7 historical reports).
- `dai/.claude/skills/dai-next-slice-planner/SKILL.md` -- freshness check, hard eligibility
  gates, <=3 ranked candidates across nine documented dimensions, exactly one recommendation,
  alternatives explained, one gated operator decision, Slice Synopsis final. Hard
  non-responsibilities: never creates/authorizes/schedules/executes, never writes dates.
- `dai-vault/06 Execution/plans/platform-delivery-timeline-v1.md` -- operator-owned intent:
  authorization posture block (currently no-spend, all three not-authorized, sourced) + four
  initiatives seeded from WI-0006 deferred items and WI-0002/0003 backlog. All date fields
  deliberately empty: no operator dates exist, and the system may not invent them.
- `dai/.gitignore` -- `scripts/dev/planning/out/` (live snapshots are ephemeral).

## 2. verification

- Fixture suite: **45 passed / 0 failed** (`scripts/dev/planning/test-build-next-slice-snapshot.ps1`;
  fixtures model malformed front matter, duplicate ids, both disposition conventions,
  adversarial mtime, timeline pathologies, fail-closed authorization, determinism, read-only).
- Real-state run: WI-0004..0007 integrated, 0 integration continuations, no-spend posture,
  gamePk capture boundary surfaced as a deferred candidate, runtime cold, 1 non-material
  warning. The real run caught a fixture-blind gap (pre-WI-0007 WIs record integration in the
  links block, not a final-disposition heading) -- fixed red-first, suite extended to cover it.
- Planner desk-run on the real snapshot: WI-0002/0003 correctly gated out (activation gate;
  unsatisfied `not_before` + dependency); 3 ranked; recommendation = **gamePk propagation
  (proposed WI-0009)**, earned by the `6c9d433e` exclusion evidence and capability-unblock
  value; decision request left gated. All 13 planner behavior checks hold.
- Review: APPROVE, zero blockers. Guardrails: no paid calls, no writes to any operational
  system, diff confined to scoped files, phantom + vault untracked files preserved, runtime
  never started.

## 3. repo state

- before: dai `main` @ `41e0a46` == origin; vault `main` @ `e7e5c5e` == origin.
- after: dai `wi/0008-evidence-grounded-next-slice-planner` 1 local commit, not pushed;
  vault `main` 1 local commit, not pushed. Hashes in the WI links block via the final report.

## 4. known limitations / open issues

- WI-0001 `integrationState` reads `unknown` at file level (integration recorded only in the
  MOC); a MOC collector is a named future seam. Not material: `implementationState=complete`
  keeps it out of pending candidates.
- Reconciliation metrics are pointer-only by design; a structured readout collector is a
  future seam.

## 5. deferred work (each item separate; none authorized, none started)

1. WI-0008 integration and push.
2. Operator decision on proposed WI-0009: propagate `gamePk` through
   `CompetitionMatchupInput` (the planner's gated recommendation from the real-state run --
   an existing proposal to continue, not a new WI to mint again).
3. MOC-based integration collector for older WIs whose integration state exists only in the
   MOC (the WI-0001 `unknown` limitation in section 4).
4. Structured reconciliation-readout collector (today a provenance-carrying pointer).
5. Future planning seams documented in WI-0008: product or tenant scopes, recurring reports,
   cost estimates, tracker adapters, recommendation feedback, candidate-to-WI drafting.
6. WI-0002 (backlog, operator activation gate).
7. WI-0003 (backlog, gated on a concrete second consumer).

Operator decision from the real-state planner run, left gated:
`Authorize creation and execution of WI-0009: gamePk Propagation Through the Generation
Request v1` -- or authorize WI-0008 integration first.

### Slice Synopsis

**Change:** Completed the evidence-grounded planning system locally: deterministic snapshot
tooling, `dai-next-slice-planner`, and the operator-owned delivery timeline.
**Reason:** Next-slice selection lacked deterministic evidence assembly, fail-closed
authorization, and separation between operator dates and system proposals.
**Proof:** 45/45 assertions passed; the real-state run recognized WI-0004 through WI-0007 as
integrated and produced one evidence-linked, gated recommendation.
**State:** `dai` at `88c9f09` and vault at the corrected documentation HEAD, local only;
nothing pushed; WI-0008 implementation complete; runtime cold.
**Next:** Separately authorize WI-0008 integration and push.
