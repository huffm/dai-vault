---
title: "WI-0021 Protocol Hardening Candidate Specifications v1"
type: "plan"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0021 AI Engineering Protocol Hardening Candidate Specifications v1"
repos:
  dai: "unchanged (read-only inspection at 85a8831)"
  dai-vault: "docs-only (WI-0021 branch; integrated -- ff to dai-vault/main, branch retained at 361a2e6 per final disposition; front matter corrected 2026-07-24 audit)"
tags:
  - system-development
  - work-item
related:
  - "06 Execution/plans/protocol-hardening-candidate-specifications-v1.md"
  - "06 Execution/plans/hardening-ready-queue-v1.md"
  - "06 Execution/plans/ai-engineering-hardening-catalog-v1.md"
---

# WI-0021 protocol hardening candidate specifications v1

## problem  <!-- LITE -->

The six operator-approved hardening candidates (PH-01..PH-06) existed only as queue
cards and catalog rows; an implementer pulling one would have to invent scope,
acceptance criteria, fixtures, and RC-impact treatment mid-branch -- the exact
failure modes (scope creep, criteria-after-implementation, silent RC change) the
hardening program forbids.

## desired behavior

A future engineer or agent pulls exactly one candidate and finds: verified loci,
observed-vs-preventive evidence, falsifiable acceptance criteria, Inspect/Prove/Guard
decomposition, lane + RC classification, idle-window tasks, and named unresolved
decisions -- without reopening the architecture or inferring authority.

## affected surfaces

NEW 06 Execution/plans/protocol-hardening-candidate-specifications-v1.md (authoritative);
ready-queue PH table + catalog section 1b (links only); this WI; current-slice entry.

## non-goals

No implementation; no implementation WIs or branches (all slugs use <next-id>); no
protocol vocabulary change (G-01 untouched); no dormant-station activation; no RC,
G-10, R-05, commercial, cloud, or multisport authority; no new candidates beyond the
six (adjacent ideas recorded as deferred notes only).

## acceptance criteria  <!-- LITE -->

Exactly six candidates specified with all 40 required fields; acceptance criteria
falsifiable (no "tests pass"/"behavior is correct" stubs); the WI-0020 evidence
taxonomy used with achievable-vs-unproven classes stated per candidate;
Inspect/Prove/Guard present for all six and labeled as branch-execution phases, not
cognitive micro-actions; definitions of ready and done present; RC-neutral vs
RC-affecting policy stated and applied; readiness verdicts returned (PH-01/04/06
READY, PH-02/03 READY WITH OPEN QUESTION, PH-05 NOT READY with exact missing
decisions); recommended order justified; no candidate marked active or pulled.

## test plan  <!-- LITE -->

Docs-only: verification = full diff inspection, strict planning snapshot (0 warnings,
actual continuations recorded), cross-reference consistency with catalog/queue.

## implementation notes

Specifications grounded in the 2026-07-15 read-only inventories at dai 85a8831
(protocol implementation + discipline controls) plus targeted re-verification;
per-candidate loci cite file:line evidence recorded in the maturity matrix. Perceive
intake guidance labeled noncanonical (Observe/Frame/Bound permitted labels only).

## docs to update

Spec doc (new), ready queue, catalog, current-slice, this WI.

## verification commands  <!-- LITE -->

git diff --stat (branch vs main); pwsh dai/scripts/dev/planning/
build-next-slice-snapshot.ps1 -Strict.

## risks

Spec drift vs code once implementation begins (mitigation: Inspect phase re-verifies
loci at pull time); PH-05 specification could read as implementation pressure
(mitigation: NOT READY verdict + hard operator-decision dependencies named).

## links  <!-- LITE -->

- work item: WI-0021
- branch: wi/0021-protocol-hardening-candidate-specifications (dai-vault; "local only,
  NOT pushed, NOT integrated" superseded 2026-07-24 audit -- pushed and integrated per
  final disposition, retained at `361a2e6`)
- pr: —
- commits: recorded at close on the branch
- tests: none (docs-only)
- verification notes: slice final report + current-slice entry
- docs updated: spec doc (new), ready queue, catalog, current-slice
- lessons: none

## final disposition (2026-07-15)

**complete + reviewed + integrated.** Specification commit `e3a5eb3` reviewed against
the WI-0021 authorization (six candidates, 40 fields, falsifiable criteria, doctrine
+ taxonomy conformance); one confirmed WIP-policy inconsistency corrected in the
narrow second commit `361a2e6` ("align PH-06 sequencing with WIP policy" -- a pulled
PH-06 card counts as an active implementation branch; no implicit WIP exception;
e3a5eb3 unamended). Fast-forwarded to dai-vault/main (pure ff, tree-identical);
branch pushed and retained locally and remotely at `361a2e6`. No PH candidate
pulled; no implementation authority granted; no unresolved specification work
remains. Preserved: G-10 unexecuted; R-05 gated; G-01 unresolved; RC Gate 1
(2026-07-17) next operational event; zero spend and writes.

## final handoff requirements

Slice handoff appended to current-slice.md on the branch; integration recorded in a
separate integration-recording commit on main (this update).
