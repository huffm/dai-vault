---
title: "Capability-Selection Plan Contract Review v1 (2026-07-20)"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0031 Slice 4 r2: selection-plan contract correction, review, integration"
repos:
  dai: "code+tests (DevCore.Domain capability-selection; branch wi/0031-deterministic-ranking-and-plan-building)"
  dai-vault: "docs-only"
tags:
  - system-development
  - platform
  - capability-selection
  - review
related:
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
  - "06 Execution/reports/capability-selection-deterministic-plan-building-slice-4-2026-07-20-v1.md"
---

# capability-selection plan contract review v1

## purpose

Independent review of the unintegrated WI-0031 Slice 4 selection-plan implementation:
reproduce the confirmed contract defects before correcting, correct them in new commits
preserving the originals, align the documentation to the honest delivered semantics, verify,
and fast-forward integrate. Offline only; no model/tool/source/database/application calls.

## reproduced defects (each failed against the original tip for the stated reason)

- **A context relabeling.** `DeterministicPlanBuilder.Build` took a separate
  `ResolutionContext`, so a result produced under one tenant could be relabeled with a
  different tenant/context (the builder's required 4th `context` argument is the seam).
- **B validation bypass + mutable source.** `WeightProfile` was a `record` with a public
  constructor: `new WeightProfile("x","1", {"mystery":1.0})` compiled and succeeded, and the
  caller's mutable dictionary backed the profile.
- **C non-reproducible / unexplainable.** The plan recorded only a profile name/version and
  raw scores; two different weight sets with the same name/version were indistinguishable and
  weighted contributions could not be reconstructed from the trace.
- **D duplicate-component last-write-wins.** Duplicate known components were merged with
  last-value-wins, so reordering equivalent duplicate facts changed the score (proven:
  `[reliability=0.1, reliability=0.9]` scored differently from the reverse).
- **E canonical-relevance overwrite.** A `semantic_relevance` score component overwrote the
  canonical relevance field (proven: relevance 0.5 + component 0.9 scored 0.9).
- **F weak isolation test.** The original tenant-isolation test never exercised a
  result/context mismatch.
- **G documentation overstatement.** Docs implied reproducible recipe planning and called a
  test-only profile a delivered profile.

## corrections (new commit dai `209d485`; original `f926484` preserved)

- **context ownership:** `ResolutionResult` owns the immutable `ResolutionContext`; `Build`
  takes no separate context; tenant/role/node/registry/policy come only from the result and
  the full context is serialized. No caller relabeling seam remains.
- **weight-profile integrity:** factory-only (`TryCreate`) and immutable; no public
  constructor; defensive ordered copy; mutating caller collections cannot alter the profile;
  rejects blank name/version, duplicate, unknown, missing-required, non-finite, and negative
  weights (zero valid); persists ordered weights + a deterministic content SHA-256.
- **candidate score integrity:** blank / duplicate / reserved(`semantic_relevance`) /
  out-of-[0,1] known-favorable components are rejected deterministically at the resolver
  boundary; no last-write-wins; the canonical relevance is never overwritten; unknown
  components remain non-authoritative (dropped, never favorable).
- **explainable scoring:** each eligible candidate records `ScoreContribution`s (component,
  normalized value, weight, weighted contribution) reconciling with the final score at the
  rounding precision; blocked/pending candidates have none.
- **artifact semantics:** plan schema `capability-selection-plan/1.0`; the output is a
  ranked, bounded, non-executable selection plan (Stage E partial), not a Stage F recipe
  compiler; all-false authority ledger; no credential/payload/argument/callback/endpoint/DI/
  CLI/scheduler/persistence/telemetry/model/gateway/execution; unused code and fixture params
  removed; comments lowercase ascii.

## exact context / weight / score semantics

- **context:** a plan's tenant/role/protocol node/registry version/policy version are the
  result's owned immutable context; there is no relabeling parameter.
- **weights:** zero is valid; negative, non-finite, blank/duplicate/unknown names, and
  missing required components are rejected; the content SHA-256 makes same-name/version but
  different-weight profiles distinguishable.
- **score contributions:** `weighted_contribution = round(weight * normalized_value)`; the
  final score = `round(sum(weighted_contribution))` at 12 decimals; absent facts use the
  0.0 floor; unknown candidate components contribute nothing.

## why a selection plan, not a recipe compiler

The delivered artifact ranks eligible candidates and proposes a bounded, ordered,
non-executable plan for review (Stage E ranking, partially delivered). It contains no
dependency graph, input/output binding, retry, ceiling, stop condition, fallback, or
idempotency compilation -- that is Stage F, which remains deferred. The plan executes
nothing and grants no permission; the Tool Gateway remains the runtime permission authority.

## verification

Targeted capability-selection + plan tests 57/57. Full DevCore.Api.Tests **1416 passed / 0
failed** (was 1409). DevCore.Domain build 0 warnings (only the pre-existing NU1903 package
advisory elsewhere; 0 csproj changes on the branch). A real Slice-3 projection -> resolver
-> builder composition is deterministic and honestly ends `authorization_pending` (only the
protocol-node dimension is enforceable today, so fail-closed holds end to end). Fresh-process
determinism reconfirmed across two independent test processes; the fixed-seed corpus now
includes context and score-component reorder cases; `git diff --check` clean;
secret/machine-path/authority scans clean; strict snapshot 24 WIs / 0 warnings; protected/
classified drift byte-identical. Code review ran against the full branch range.

## next step

Independent review + integration are completed in this slice (fast-forward, both repos).
WI-0031 Slice 5 (execution telemetry / offline evaluation) and Slice 6 (governed pilot)
remain deferred; no production weight selection and no recipe compiler are authorized here.
