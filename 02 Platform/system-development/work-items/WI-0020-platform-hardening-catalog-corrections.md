---
title: "WI-0020 Platform Hardening Catalog Corrections v1.1"
type: "plan"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "DAI AI Engineering Hardening Catalog v1.1 -- Evidence Taxonomy, Branch Policy, Credential Exposure, and Fitness-Check Corrections"
repos:
  dai: "unchanged (read-only history verification only)"
  dai-vault: "docs-only (dedicated branch, NOT integrated)"
tags:
  - system-development
  - work-item
related:
  - "06 Execution/plans/ai-engineering-hardening-catalog-v1.md"
  - "06 Execution/reports/protocol-coverage-and-maturity-matrix-v1.md"
  - "06 Execution/plans/hardening-ready-queue-v1.md"
  - "06 Execution/plans/ai-engineering-fitness-checks-v1.md"
---

# WI-0020 platform hardening catalog corrections v1.1

## problem  <!-- LITE -->

The 2026-07-15 hardening catalog v1 shipped with four review defects: (1) evidence
language mixed non-equivalent claims ("buyer proven" for a never-buyer-used surface;
"paid-run proven" for prose whose semantics are unevaluated); (2) the queue's Green
lane said vault-docs cards "commit direct to vault main," contradicting the
branch-on-pull rule; (3) G-10 read as if a documentation checklist remediates
credential exposure, and asserted a FALSE fact (appsettings.Development.json is
gitignored/untracked with zero history -- verified read-only; the tracked
appsettings.json historically carried the SQL connection string until `ded9969`);
(4) the fitness-check total was ambiguous between 14 adopted and 3 conditional.

## desired behavior

One canonical evidence taxonomy used consistently; every pulled card branch-based with
integration separate; G-10 = classification + preparation with explicit R-05
escalation, never remediation; one authoritative count: 14 adopted = 11 existing + 3
added, conditionals excluded.

## affected surfaces

06 Execution/plans/ai-engineering-hardening-catalog-v1.md;
06 Execution/reports/protocol-coverage-and-maturity-matrix-v1.md;
06 Execution/plans/hardening-ready-queue-v1.md;
06 Execution/plans/ai-engineering-fitness-checks-v1.md;
06 Execution/handoffs/current-slice.md (append-only entry).

## non-goals

No runtime/feature work; no credential rotation or revocation; no dai change; no G-01
resolution; no queue-priority change beyond corrected wording; no v2 taxonomy; no new
catalog features; no integration to main (separate authorization).

## acceptance criteria  <!-- LITE -->

Taxonomy defined once (matrix) and referenced; no contradictory "proven" claims;
deterministic-vs-semantic maturity split stated; every pulled card branch-based; G-10
classification gate + R-05 escalation explicit and not describable as remediation;
14 = 11 + 3 everywhere; conditionals labeled FC-C1..C3 not-adopted; dai untouched;
zero spend/writes.

## test plan  <!-- LITE -->

Docs-only: proof = repository-wide textual reconciliation (grep sweeps for proven /
mature / directly to main / G-10 / R-05 / rotation / remediated / 14 adopted / 11
EXIST / 3 ADD / conditional / replicas / rollback / CI) + strict planning snapshot 0
warnings / [] continuations. No runtime tests.

## implementation notes

Corrections applied smallest-first per the catalog's Inspect -> Prove -> Guard
convention; correction inventory produced before editing (session scratchpad).
Credential facts verified read-only with key names only (git ls-files, git log,
.gitignore rule, ded9969 diff names).

## docs to update

The four hardening docs (retitled v1.1 in front matter; filenames unchanged to
preserve links) + this WI + current-slice append.

## verification commands  <!-- LITE -->

git -C dai-vault diff --stat (branch vs main); grep sweeps listed in test plan;
pwsh dai/scripts/dev/planning/build-next-slice-snapshot.ps1 -Strict (0 warnings, []).

## risks

Historical current-slice entries retain the superseded wording (append-only log --
deliberately NOT rewritten; the new entry records the supersession). Readers of old
entries must prefer the v1.1 docs.

## links  <!-- LITE -->

- work item: WI-0020
- branch: wi/0020-platform-hardening-catalog-corrections (dai-vault; pushed + retained
  locally and remotely at 5c2500b)
- pr: — (none; ff-only integration, separately authorized 2026-07-15)
- commits: 5c2500b (correction commit, unamended) + the integration-recording commit
- tests: none (docs-only; snapshot + grep proof)
- verification notes: in the slice final report + current-slice entries
- docs updated: the four v1.1 hardening docs + current-slice + MOC
- lessons: append-only handoff wording cannot be edited -- supersede via new entry

## final disposition (2026-07-15)

**complete + reviewed + integrated.** Correction commit `5c2500b` reviewed against the
four correction requirements and fast-forwarded to dai-vault/main under the separate
integration authorization (pure ff, tree-identical, no merge commit, no rebase, no
amend). Branch retained locally and remotely. No correction work remains open.
Preserved: G-10 unexecuted; R-05 gated; G-01 unresolved; RC boundary unchanged
(Gate 1 2026-07-17 next); no credential value accessed; no credential remediation
performed; zero spend and writes.

## final handoff requirements

Slice handoff appended to `06 Execution/handoffs/current-slice.md` on the branch;
integration recorded in a separate integration-recording commit on main (this update).
