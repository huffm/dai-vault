---
title: "WI-0022 Representative Protocol Failure Corpus v1 (PH-01 Green subset)"
type: "plan"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0022 Representative Protocol Failure Corpus v1"
repos:
  dai: "tests-only on branch wi/0022-discern-stress-protocol-failure-corpus (NOT integrated; main frozen 85a8831)"
  dai-vault: "docs-only (coordinated branch; NOT integrated)"
tags:
  - system-development
  - work-item
related:
  - "06 Execution/reports/protocol-failure-corpus-results-v1.md"
  - "06 Execution/plans/protocol-hardening-candidate-specifications-v1.md"
---

# WI-0022 representative protocol failure corpus v1

## problem  <!-- LITE -->

Failure knowledge (4 excluded lean-mismatch runs, 823357, 823613/824818 identity
collisions, v7c thin residue, WI-0005 cache poisoning, WI-0013 unpriced metering) was
scattered across suites and history with no single deterministic corpus; regressions
in failure handling were only detectable piecemeal, and missing guards were
indistinguishable from unexercised ones.

## desired behavior

One canonical, deterministic, machine-readable corpus of 15 failure classes, each
classified guard_verified / behavior_characterized / guard_missing / not_applicable /
policy_blocked from repository evidence, exercising REAL production seams, with
explicit follow-up ownership for every gap.

## affected surfaces

dai (tests only): DevCore.Api.Tests/ProtocolFailureCorpus/{ProtocolFailureCorpus.cs,
ProtocolFailureCorpusTests.cs, ProtocolFailureSeamTests.cs};
services/agent-service/tests/test_protocol_failure_corpus_seams.py.
dai-vault: this WI, protocol-failure-corpus-results-v1.md, ready-queue PH-01 state,
current-slice handoff.

## non-goals

No runtime/endpoint/prompt/model/scoring/policy/schema/config change; no abstention
enforcement; no delivery/entitlement implementation; no sixteenth canonical class; no
implementation WIs for discovered gaps (mapped to PH-02..PH-06); no integration or
push (separately authorized).

## acceptance criteria  <!-- LITE -->

All PH-01 Green-subset criteria evaluated (WI-0022 authorization sections 10-16):
15/15 canonical families; full fixture contract incl. origin + status + seam
coverage; deterministic byte-stable serialization; offline (no network/model/db/
secrets); follow-up ownership for guard_missing + policy_blocked; unresolved-decision
naming for policy_blocked; no-runtime-claim proof for not_applicable; buyer-language
safety via the real BuyerCopySafety seam; sixteenth-class manifest check;
unknown-version fail-closed. RESULT: all satisfied -- evidence report section 4.

## test plan  <!-- LITE -->

16 harness tests + 8 C# seam tests + 3 python seam tests (focused first), then full
regressions. RESULT: focused 27/27 C# + 3/3 py; FULL 1262/1262 C# (+27) and 456/456
py (+3); vitest not run (no Angular change).

## implementation notes

Inspect re-verified WI-0021 loci at 85a8831 (one targeted correction: fixtures in
this repo are inline raw strings; no Content-copy convention -- corpus therefore
lives as static C# data, keeping csproj untouched). Seam tests mirror existing test
construction patterns verbatim (classifier helpers, DI-built gateway, projection
builders). Classification notes: PF-09
guard_verified as a VISIBILITY guard only (spend-blocking = PH-03/R-04). REVIEW
CORRECTION (integration review, 2026-07-15): PF-01 reclassified guard_verified ->
behavior_characterized -- the readiness classifier is fail-closed classification, but
the paid creation path never consults it (enforcement is procedural), matching the
authorization's provisional grouping after all; PF-06/PF-07 claims narrowed to their
exercised seams (settlement write path only; parse-time clamp).

## docs to update

Evidence report (new), ready queue PH-01 state, current-slice, this WI.

## verification commands  <!-- LITE -->

dotnet test DevCore.Api.Tests --filter "FullyQualifiedName~ProtocolFailureCorpus";
dotnet test DevCore.Api.Tests; pytest tests; git diff 85a8831 -- <production trees>
(empty); strict planning snapshot.

## risks

Corpus drift vs runtime once PH-02/PH-03/PH-04 land (mitigation: their Guard phases
must update affected fixture classifications); characterization results could be
misread as endorsed policy (mitigation: behavior_characterized explicitly means "not
proven final policy").

## links  <!-- LITE -->

- work item: WI-0022 (implements PH-01 Green subset; PH-01 marked pulled+active)
- branch: wi/0022-discern-stress-protocol-failure-corpus (dai from 85a8831 +
  coordinated dai-vault from 6ac892c; BOTH LOCAL-ONLY, not pushed, not integrated)
- pr: —
- commits: dai e348310 (contract+fixtures), d35de5c (harness), f057a39 (seam tests);
  vault commits recorded at close
- tests: +27 C# / +3 python, all green; full suites green
- verification notes: protocol-failure-corpus-results-v1.md
- docs updated: evidence report, ready queue, current-slice, this WI
- lessons: repo has no file-fixture convention -- static-data corpora keep test
  projects config-free

## final disposition (2026-07-15)

**complete + reviewed + integrated.** Integration review audited every
guard_verified classification against its exercised seam; corrections applied as
new commits (dai `a0ca54d` narrows PF-01 -> behavior_characterized and scopes
PF-06/PF-07 claims; vault `b7c6842` aligns the evidence report; final distribution
9/3/1/1/1). All suites re-verified green post-correction (1262 C# / 456 py / 27+3
focused). RC-neutrality proven (empty production diff, project-graph proof, runtime
smoke health-only cold-to-cold) and recorded in
`rc-equivalence-wi-0022-2026-07-15-v1.md`: dai/main fast-forwarded 85a8831 ->
`a0ca54d` (the new repository RC commit; same release candidate artifact); vault/main
fast-forwarded to `b7c6842` + integration record. Both branches pushed and retained.
Evidence class achieved: contract-represented + fixture-proven; nothing higher.
PH-01 CLOSED. WIP = 0. G-10 unexecuted; R-05 gated; RC Gate 1 (2026-07-17) remains
the next operational event with the opening check updated to a0ca54d.
