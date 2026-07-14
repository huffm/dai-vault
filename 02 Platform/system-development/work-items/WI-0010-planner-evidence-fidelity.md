---
title: "WI-0010 Planner Evidence Fidelity v1.1"
type: "plan"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0010 Planner Evidence Fidelity v1.1"
repos:
  dai: "code (planning tooling + planner skill compatibility)"
  dai-vault: "docs-only (incl. authorized operator timeline correction + readout sidecar)"
tags:
  - system-development
  - work-item
  - planning
related:
  - "02 Platform/system-development/work-items/WI-0008-evidence-grounded-next-slice-planner.md"
  - "06 Execution/plans/platform-delivery-timeline-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-v2-day2-2026-07-11-v1.md"
  - "06 Execution/reports/hardened-regime-baseline-measurement-2026-07-11-v1.md"
---

# wi-0010 planner evidence fidelity v1.1

**Slice type:** platform/workflow development. **Opened:** 2026-07-13.

## problem (four defects reproduced by the first post-WI-0009 planning run)

A. **MOC-only integration state reads unknown.** WI-0001's integration exists only in the MOC;
   the collector reads WI files + handoffs, so WI-0001 = `integ=unknown` (false unknown).
B. **Delivered capability still listed deferred.** WI-0006's "propagate gamePk" bullet remains
   an active candidate although WI-0009 delivered and integrated it (`d493f84`).
C. **Duplicate candidates.** The identity-status capability appears twice (WI-0006 and WI-0009
   deferred sections), undeduplicated.
D. **Reconciliation is pointer-only.** Every snapshot emits the standing warning; settled/
   excluded/Gate/discrimination/attribution facts are not machine-readable for the planner.

## source-precedence boundary (MOC fallback)

Integration evidence precedence: 1 verified live repo state; 2 structured WI lifecycle +
explicit disposition; 3 final integrated handoff; 4 **MOC integration record (new fallback)**;
5 timeline/planning reference; 6 earlier implementation handoff; 7 unknown. The MOC is
consulted ONLY when 2-3 yield unknown; MOC-only resolution carries `confidence: medium`; a
MOC/stronger-source contradiction preserves both facts, prefers the stronger source, and emits
a material warning. WI-0001 is NOT rewritten.

## candidate-identity contract (no fuzzy merging)

Canonical identity = operator timeline `initiative_id`. Mapping is explicit and deterministic:
a WI deferred bullet maps to an initiative iff (a) its origin WI is in the initiative's
`related_work_items` AND (b) at least one of the initiative's `aliases` is a case-insensitive
substring of the bullet. No alias match -> candidates stay separate + warning when equivalence
is plausible but unmapped. Dedup: one canonical candidate per initiative; ALL origins preserved,
deterministically sorted. Delivery: initiative `status: complete` + `delivered_by: WI-####`
(operator-maintained) -> candidate `status: delivered`, excluded from active ranking, history
retained (`origin`, `deliveredBy`). Historical WI bullets are never edited.

## reconciliation readout contract

New canonical sidecar `06 Execution/reports/reconciliation-planning-readout-v1.md`
(evidence-report) carrying a machine-readable `planning-readout` yaml block whose values are
COPIED (never computed) from the authoritative closeout docs, each with source citations. The
collector parses the sidecar (same simple-yaml machinery as the timeline; no brittle prose
parsing, no recomputed conclusions, no hard-coded values in code or real-workspace fixtures),
emits `reconciliation.facts` + provenance, and keeps the latest-report pointer. The standing
pointer warning is emitted only when the sidecar is missing or older than the newest
reconciliation record (stale check by front-matter dates). Convention: the sidecar is updated
manually at each cadence close, exactly like the authorization block.

Facts verified against sources for the initial sidecar (gate4 readout + hardened-regime
baseline + day-1/day-2 settlements, all 2026-07-10/11): captured 16, settled 15, excluded 1
(823357), correct 9, incorrect 6; gate conclusionsAllowed=false, failing
[discrimination_inverted, insufficient_market_disagreement], discrimination delta -0.1486
(inverted), coverage 0.6723 (met), market-opposed n=8 readable=false record 3/5, high-conf band
n=18 acc 0.4444, validDirectionalN 119; attribution Pass 14 / FAIL 0 / Unclear 1; cadence
ended, posture no-spend.

## operator-owned timeline correction (explicitly authorized in this WI)

Manual edit, never tooling: `gamepk-propagation` -> `status: complete`,
`delivered_by: WI-0009`, `completed: 2026-07-13`, `related_work_items: [WI-0006, WI-0009]`,
`integration_commit: d493f84`, status source = operator-confirmed integrated WI; no desired/
due/proposed date invented. `identity-status-refinement` gains `aliases` + WI-0009 in
`related_work_items` (dedup metadata only; stays `candidate`). Aliases added to both entries.

## scope

`dai/scripts/dev/planning/build-next-slice-snapshot.ps1` (+ tests/fixtures): MOC fallback,
candidate identity/dedup/delivery, sidecar readout collector, stale/conflict warnings, schema
1.0 -> 1.1 (additive: candidate status/origins/deliveredBy, reconciliation.facts+provenance,
integration evidence provenance). Minimal `dai-next-slice-planner` compatibility edit
(delivered candidates never ranked; structured recon consumed) + same-slice inventory note.
Vault: this WI, timeline correction, readout sidecar, MOC, current-slice, handoff.

## non-goals / out of scope

Ranking dimensions/weights; authorization policy; autonomous anything; planning DB/dashboard;
cost/report/tracker/tenant seams; `CompetitionMatchupInput`; identity-status redesign;
WI-0002/0003; Gate recomputation; reconciliation writes; paid calls; capture; push/merge/PR.

## acceptance criteria

1. WI-0001 resolves `integrated` via MOC fallback with provenance (path, precedence 4,
   confidence medium) -- without touching WI-0001's file.
2. Stronger evidence beats MOC; contradiction warns materially; absent everywhere -> unknown.
3. gamePk candidate classified `delivered` (origin WI-0006, deliveredBy WI-0009), excluded
   from active candidates, history retained.
4. Identity-status emits exactly one canonical candidate with both origins, deterministic
   ordering; similar-but-unmapped candidates are NOT merged and warn.
5. Timeline correction present; tooling proven read-only over it; operator dates untouched.
6. `reconciliation.facts` populated with provenance from the sidecar; no new conclusion
   computed; malformed/missing/stale sidecar fails safe (unknown + warning).
7. Snapshot deterministic for fixed inputs + AsOfUtc; schema 1.1.
8. Clean real-state run: WI-0001..0009 integrated, zero continuations, zero unexplained
   warnings, strict mode passes; planner validation run consumes the corrected evidence,
   ranks no delivered/completed work, returns exactly one gated recommendation.
9. Fixture suite green (final count reported at close, not predeclared); review APPROVE;
   local commits only; runtime cold.

## rollback

Revert the two commits; the sidecar and timeline metadata are additive documentation; schema
1.1 consumers degrade to warnings on 1.0 output. No persisted runtime data involved.

## approval boundary / integration gate

WI-0010 authorizes evidence-fidelity work only, local commits only. Integration and push are
a separately gated continuation of this WI. (Gate exercised: the continuation was authorized
and completed -- integrated and pushed as `e64567f`; see final disposition.)

## validation record (2026-07-13)

- defects reproduced pre-change on the unmodified tooling: A WI-0001 `integ=unknown`;
  B gamePk candidate active (origin WI-0006) despite WI-0009 integrated; C identity-status
  x2 (WI-0006+WI-0009); D `facts` absent + standing pointer warning.
- fixture suite: **73 passed / 0 failed** (was 45; +28 new asserts incl. two legacy asserts
  updated to the 1.1 contract). red-first throughout (19 new failures before implementation).
- real-state run (STRICT, zero warnings): WI-0001 `integrated` via moc (precedence 4,
  confidence medium) without touching its file; WI-0004..0009 integrated via their stronger
  sources; 0 continuations; gamePk candidate `delivered` (origins WI-0006, deliveredBy
  WI-0009), excluded from active; identity-status ONE canonical candidate, origins
  WI-0006+WI-0009; capture-operation `gated`, never ranked; recon facts structured with
  provenance (settled 15 / excluded 1 / 9-6 / gateAllowed false / delta -0.1486 /
  attribution 14-0-1); schema 1.1; byte-determinism retained.
- planner validation run on the corrected snapshot: no delivered/completed work ranked,
  <=3 of 4 active candidates ranked, exactly one gated recommendation, confidence high.
- review: APPROVE, zero blockers. accepted noted risk: moc parsing matches that registry's
  own bold status markers (single stable file; contradictions warn, never trusted).

## links

- work item: WI-0010 (ADO: AB#-- when wired)
- branch: `wi/0010-planner-evidence-fidelity` (dai, from `d493f84`; deleted locally and
  remotely after integration -- deviation from the branch-retained convention)
- pr: -- (not authorized)
- commits: dai `e64567f` (tooling + fixtures + planner-skill note; integrated to dai/main and
  pushed, origin synced) and dai-vault (this WI, timeline correction, readout sidecar, MOC,
  inventory, current-slice, handoff) -- vault hashes in the slice handoff at close
- tests: `scripts/dev/planning/test-build-next-slice-snapshot.ps1` -> 73/73
- verification notes: validation record above; strict real-state run zero warnings
- docs updated: this WI; MOC; `platform-delivery-timeline-v1` (authorized correction +
  operational-gated initiative); `reconciliation-planning-readout-v1` (new sidecar);
  skills inventory; current-slice; handoff
- lessons: operator-owned structured sidecars (authorization block, planning readout) beat
  prose regexes every time the evidence layer needs a new fact -- copy verbatim with
  citations, parse the copy, warn on staleness; and explicit alias+relation mapping gives
  dedup without fuzzy-merge risk.

## final disposition

Complete + integrated (2026-07-14). Integration commit dai `e64567f` == dai/main ==
origin/main, a pure fast-forward from `d493f84`; the integration and push occurred before
this documentation correction, which exists solely to reconcile the vault record with the
already-integrated repository state. Branch `wi/0010-planner-evidence-fidelity` was deleted
locally and remotely (a disclosed deviation from the branch-retained convention of
WI-0004..0009). Focused fixture suite re-verified on main 2026-07-14: 73/73. No
implementation work remains open. This supersedes the prior "implementation complete, local
only / integration separately gated" disposition of 2026-07-13.
