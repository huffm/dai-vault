---
title: "WI-0008 Evidence-Grounded Next-Slice Planner v1"
type: "plan"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0008 Evidence-Grounded Next-Slice Planner v1"
repos:
  dai: "code (planning tooling + skill layer)"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - planning
  - workflow
related:
  - "02 Platform/system-development/work-items/WI-0007-mandatory-work-items-and-slice-synopsis-workflow.md"
  - "06 Execution/plans/platform-delivery-timeline-v1.md"
  - "06 Execution/skills/dai-skills-inventory-v1.md"
  - "06 Execution/patterns/okf-documentation-review-guide-v1.md"
---

# WI-0008 evidence-grounded next-slice planner v1

**Slice type:** platform/workflow development. **Opened:** 2026-07-13.

## problem

Choosing the next slice currently requires a human (or agent) to re-read the WI spine, MOC,
current-slice tail, recent handoffs, and deferred-work lists, then reconcile status,
authorization, and sequencing by hand. Nothing assembles that evidence deterministically, and
nothing distinguishes operator-owned dates from system suggestions. Prior sessions have shown
the failure modes: stale copies outranking canonical docs (see the pasted-context false-defect
incident recorded in the operating-context memory), and completed work re-proposed as pending.

## why now

WI-0004..0007 are integrated; the spine, OKF taxonomy, and qualification gate (WI-0007) are
now stable enough to be machine-readable. The next planning decision (gamePk propagation vs
status refinement vs presentation backlog) is exactly the kind of tradeoff an evidence-grounded
planner should expose rather than a prompt asserting it.

## governing objective / intended outcome

A read-only, two-layer planning system: (1) a deterministic snapshot script assembles current
operating state -- work items, deferred candidates, handoffs, reconciliation references,
timeline intent, repo/runtime/authorization posture -- as versioned JSON where every
ranking-relevant fact carries provenance; (2) a `dai-next-slice-planner` skill consumes the
snapshot, applies hard eligibility gates, ranks at most three candidates across documented
dimensions, and recommends exactly one bounded next slice with an explicit operator decision
request. The planner never creates, authorizes, schedules, or executes work.

> Deterministic tooling assembles facts. The planning skill evaluates tradeoffs.

## scope

- `dai/scripts/dev/planning/build-next-slice-snapshot.ps1` (new) -- read-only evidence
  assembly, versioned JSON, provenance, precedence, conflict warnings, fail-closed
  authorization.
- `dai/scripts/dev/planning/test-build-next-slice-snapshot.ps1` + `fixtures/` (new) --
  deterministic fixture tests per the repo's ps test convention (dot-source guard, Assert,
  exit code).
- `dai/.claude/skills/dai-next-slice-planner/SKILL.md` (new) -- planning evaluation layer.
- `dai/.gitignore` -- ignore the generated snapshot output folder.
- `dai-vault/06 Execution/plans/platform-delivery-timeline-v1.md` (new) -- operator-owned
  delivery timeline (type "plan"); seeds only evidence-backed initiatives; no invented dates.
- vault records: this WI, MOC, skills inventory (same-slice rule), current-slice, handoff.

## out of scope

Paid calls; capture; reconciliation/outcome/exclusion/calibration writes; Gate changes;
automatic WI creation/scheduling/execution; changing operator dates; decision/confidence/
scoring/market/buyer/prompt behavior; migrations; a planning database; a dashboard; vector
indexing; multi-agent planning debate; WI-0002/0003 implementation;
`CompetitionMatchupInput.gamePk`; push/merge/PR/rebase/amend.

## architecture / ownership boundary

Two layers, one owner per concern: the **snapshot contract** (schema + collection semantics)
is owned by the tooling under `scripts/dev/planning/`; **planning evaluation** by
`dai-next-slice-planner`; **timeline intent** by the operator-owned timeline doc; **work-item
lifecycle** stays with WI-0007 governance; **handoff + Slice Synopsis** stays with
`dai-agent-handoff`. The skill consumes the snapshot instead of re-scanning the vault; the
script assembles facts and never recommends.

## key decisions

1. **No reuse candidates exist** -- inventoried `scripts/` (start/stop, sports ops, api-safe
   test runner): no front-matter, MOC, planning, or JSON-schema utilities present. The
   snapshot script is new code; the reused assets are the ps test convention
   (`test-stop-platform-api.ps1` pattern), script header conventions, and the OKF taxonomy.
2. **Authorization posture is operator-owned and fail-closed.** The timeline doc carries an
   explicit `authorization` block (paid calls / capture / reconciliation writes) with source
   references; the script reads it verbatim. Anything absent is reported `unknown`, never
   inferred from silence.
3. **Simple-YAML subset, no new dependency.** Front matter and timeline entries are parsed by
   a small line-based parser (scalars + string lists + one nesting level) rather than adding
   the powershell-yaml module. The vault's canonical 9-field schema fits this subset; anything
   unparseable degrades to a warning, never a crash.
4. **Front-matter dates outrank file mtime.** Handoff recency and WI lifecycle come from
   front-matter `date:` plus explicit disposition text, never filesystem timestamps.
5. **Determinism contract:** with `-AsOfUtc` supplied, `generatedAtUtc == asOfUtc` and output
   is byte-identical for fixed inputs (tested).
6. **Generated snapshots are ephemeral** -- default output under
   `scripts/dev/planning/out/` which is git-ignored; only fixtures are committed.
7. **Disposition mapping:** front-matter status stays the canonical lowercase set; the script
   derives `implementationState`/`integrationState` from the WI's final-disposition/links text
   (`integrated`, `implementation complete ... integration separately gated`, etc.), so
   "complete" WIs are never pending candidates and unintegrated work surfaces as an
   integration continuation (WI-0007 status discipline, machine-applied).

## risks

- fragile regex/heuristic extraction from prose -> mitigated: extract only from structured
  front matter, explicit headings (`## deferred`, `## final disposition`), and the structured
  timeline; everything else becomes a pointer + warning, not a parsed "fact";
- stale snapshot consumed as current -> planner must check `generatedAtUtc`/`asOfUtc`
  freshness first and lower confidence when stale;
- planner drift into authority -> non-responsibilities enumerated in the skill; output ends
  with an operator decision request, never an action;
- scope creep toward frameworks/databases -> explicitly out of scope; v1 is one script + one
  skill + one doc.

## guardrails

Read-only collection (tested: no writes outside the output path; timeline file hash unchanged
after a run). Local commits only; nothing pushed. Runtime cold start/end.

## acceptance criteria

1. Snapshot script produces schemaVersion 1.0 JSON with the agreed top-level shape; every
   ranking-relevant fact carries provenance (source path, type, date, precedence, confidence).
2. Completed+integrated WIs are excluded from pending candidates; implementation-complete but
   unintegrated work is emitted as an integration continuation.
3. Deferred candidates remain distinct from formal WIs and carry their originating source.
4. Latest authoritative handoff selected by front-matter date + integration marker, not mtime.
5. Missing authorization evidence -> `unknown` (fail-closed), with source refs when present.
6. Timeline semantics preserved: `desired_by`/`due_by`/`not_before` distinct;
   `proposed_by_system` never becomes an operator date; missing `date_source` warns;
   dependency cycles and `not_before`>`due_by` produce warnings; the script never mutates the
   timeline.
7. Material conflicts (duplicate WI ids; integrated WI listed as pending in the timeline)
   produce evidence warnings and both facts preserved.
8. Deterministic output for fixed inputs + fixed `-AsOfUtc` (byte-identical).
9. `dai-next-slice-planner` applies the hard gates, ranks <=3, recommends exactly 1, explains
   alternatives, states the operator decision, ends with the Slice Synopsis, and never
   creates/authorizes/schedules work.
10. Real-state run: WI-0004..0007 recognized complete+integrated; no pending integration
    continuation; no-spend/capture posture surfaced; gamePk capture boundary visible as a
    deferred candidate; planner produces an evidence-linked recommendation.
11. Fixture test suite green; code review APPROVE; local commits only; runtime ends cold.

## verification plan

`pwsh scripts/dev/planning/test-build-next-slice-snapshot.ps1` (fixtures under
`scripts/dev/planning/fixtures/`); real-state snapshot run + planner desk-run recorded in the
WI validation record; guardrail greps; final cold check.

## rollback / reversibility

Delete the two commits: no runtime, schema, persisted-data, or behavioral surface is touched;
the planner is additive tooling + doctrine.

## dependencies / prerequisites

WI-0007 governance (integrated, live on main). None else.

## approval boundary

WI-0008 authorizes the read-only planning system described here only. The planner's
recommendations are advisory; work-item creation, authorization, scheduling, dates, and
execution remain operator-owned. No paid calls, no operational sports actions, no pushes.

## deferred work

Future seams (explicitly not v1): more collectors, product/tenant scopes, recurring reports,
cost estimates, tracker adapters, operator feedback loop, candidate-to-WI drafting.

## validation record (2026-07-13)

- fixture suite: `pwsh scripts/dev/planning/test-build-next-slice-snapshot.ps1` -> **45
  passed / 0 failed** (front matter incl. malformed + duplicates; disposition mapping incl.
  the links-block convention of pre-WI-0007 WIs; deferred-vs-WI distinction; handoff
  selection by front-matter date under adversarial mtime; timeline field separation,
  cycle/impossible-window/provenance warnings; fail-closed authorization; byte-determinism
  under fixed `-AsOfUtc`; material-conflict warning; read-only guarantees via fixture-count
  + timeline-hash equality).
- one gap found by the REAL-state run and fixed red-first: links-block integration was
  undetected, so WI-0004/0005/0006 read `unknown`; after the fix they read `integrated`.
- real-state run: 8 WIs (0004..0007 integrated; 0002/0003 blocked; 0008 in-progress);
  0 integration continuations; authorization no-spend (all three not-authorized, sourced);
  5 deferred candidates incl. the gamePk capture boundary; 4 timeline entries, all dates
  honestly empty; runtime cold; 1 non-material warning (reconciliation pointer).
- planner desk-run on the real snapshot: ineligibles (WI-0002 activation gate, WI-0003
  not_before) reported unranked; 3 ranked; exactly 1 recommendation (gamePk propagation --
  earned by the `6c9d433e` exclusion evidence + capability unblock, not hard-coded);
  decision request left gated. All 13 planner behavior checks hold.
- known limitation: WI-0001 integrationState = `unknown` at file level (its integration is
  recorded only in the MOC); a MOC collector is a named future seam.

## links

- work item: WI-0008 (ADO: AB#-- when wired)
- branch: `wi/0008-evidence-grounded-next-slice-planner` (dai, from `41e0a46`) -- local
  only, not pushed
- pr: -- (not authorized)
- commits: recorded in the slice handoff at close (dai: tooling+skill+tests; dai-vault:
  WI + timeline + records)
- tests: `scripts/dev/planning/test-build-next-slice-snapshot.ps1` (45 asserts, fixtures
  under `scripts/dev/planning/fixtures/`)
- verification notes: validation record above; diff confined to scoped files (only
  platform-tree entry in status is the pre-existing csproj phantom)
- docs updated: this WI; MOC; `06 Execution/plans/platform-delivery-timeline-v1.md` (new);
  skills inventory (same-slice rule); current-slice; slice handoff
- lessons: (1) fixture tests alone missed a real-vault convention (links-block integration);
  running collectors against the REAL corpus is part of the test plan for evidence tooling,
  not an afterthought. (2) honest `unknown` + a pointer beats a fragile prose regex -- the
  planner can lower confidence on `unknown`, but a silently wrong "fact" would poison
  ranking. (3) operator-owned intent (dates, authorization) belongs in one operator document
  the tooling reads verbatim and may never write.

## final disposition

Implementation complete, local only (2026-07-13). Integration and push separately gated.
