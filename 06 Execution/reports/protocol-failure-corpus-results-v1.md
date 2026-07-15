---
title: "Protocol Failure Corpus Results v1 (WI-0022, 2026-07-15)"
type: "evidence-report"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "WI-0022 Representative Protocol Failure Corpus v1 (PH-01 Green subset)"
repos:
  dai: "tests-only branch wi/0022-discern-stress-protocol-failure-corpus (NOT integrated)"
  dai-vault: "docs-only (coordinated WI-0022 branch)"
tags:
  - evidence
  - hardening
  - protocol
  - testing
related:
  - "06 Execution/plans/protocol-hardening-candidate-specifications-v1.md"
  - "02 Platform/system-development/work-items/WI-0022-representative-protocol-failure-corpus.md"
---

# protocol failure corpus results v1

Deterministic failure corpus for the 15 canonical classes (PH-01 Green subset).
EVIDENCE CLASS ACHIEVED: **contract-represented + fixture-proven** for the corpus
contract, plus characterization of existing production seams. NOT achieved (by
design): paid-run observed, new production-observed, operationally proven,
commercially validated. A green harness is NOT proof of runtime safety.

## 1. fixture manifest (15/15; schema 1.0; corpus in
`dai/platform/dotnet/DevCore.Api.Tests/ProtocolFailureCorpus/`)

| id | class | origin | status | seam coverage | follow-up |
|---|---|---|---|---|---|
| PF-01 | missing required evidence | observed | behavior_characterized (REVIEW-CORRECTED: the paid creation path never consults readiness -- classification/reporting is fail-closed, enforcement is procedural) | existing production seam (SourceReadinessClassifier) | PH-03 (missing-required-evidence enforcement + block-vs-warn) |
| PF-02 | stale required evidence | preventive | policy_blocked | policy unresolved (no freshness state exists) | PH-02 + PH-03 |
| PF-03 | unavailable provider | observed (WI-0005 incident) | behavior_characterized | existing production seam (absence collapses to missing; unavailable != missing NOT distinguishable) | PH-02 + G-06 design |
| PF-04 | contradictory evidence | preventive | guard_missing | runtime seam absent (no cross-source contradiction detector) | PH-02 + PH-03 |
| PF-05 | requested/resolved identity mismatch | reconstructed (WI-0006 fixtures; 823613 collision) | guard_verified | existing production seam (ambiguity fails closed w/ candidates; mismatch selects nothing) | — |
| PF-06 | incomplete provenance | observed (v7c thin residue) | guard_verified (SETTLEMENT WRITE PATH ONLY; generation-side attribution gaps warn-only) | existing production seam (SettlementProvenance -> 422) | — |
| PF-07 | unsupported directional position | preventive | guard_verified (parse-time clamp; null-posture projection covered by existing WI-0011 no-position suites) | existing production seam (python posture clamp to None) | — |
| PF-08 | unresolved decision contradiction | observed (4 excluded mismatch runs) | guard_verified | existing production seam (PotentialMismatch detection; settlement 422) | PH-03 (earlier blocking), PH-04 |
| PF-09 | unknown model pricing | observed (WI-0013 pre-fix) | guard_verified | existing production seam (named unpriced_model state; VISIBILITY guard -- execution not blocked) | PH-03 posture; R-04 gated |
| PF-10 | unauthorized tool action | preventive | guard_verified | existing production seam (ToolGateway fail-closed x2) | PH-06 |
| PF-11 | duplicate active identity | observed (823613; 824818) | guard_verified | existing production seam (DuplicateRunGuard matrix) | WI-0015 (multi-instance, gated) |
| PF-12 | buyer projection from incomplete residue | observed (823357 honest render) | behavior_characterized | existing production seam (honest recap states; MALFORMED residue validation missing) | PH-04 |
| PF-13 | numeric-confidence leakage | preventive | guard_verified | existing production seam (buyer contracts structurally confidence-free) | — |
| PF-14 | unsupported profitability/superiority claim | preventive | guard_verified | existing production seam (BuyerCopySafety fail-closed suppression) | — |
| PF-15 | delivery without valid entitlement | preventive | not_applicable | runtime seam absent (delivery is manual; runbook 7 + ledger contract preserved as expected future contract) | PH-05 (NOT READY) |

Totals (REVIEWED 2026-07-15): guard_verified 9 | behavior_characterized 3 |
guard_missing 1 | policy_blocked 1 | not_applicable 1. Origins: observed 6 |
reconstructed 1 | preventive 8. Review corrections: PF-01 reclassified
guard_verified -> behavior_characterized (creation path bypass); PF-06/PF-07
claims narrowed to their exercised seams.

## 2. harness architecture + anti-duplication

Corpus = static, version-controlled C# fixture data (schema 1.0) + integrity harness
(`ProtocolFailureCorpusTests`, 16 tests): manifest completeness (exactly 15;
sixteenth-class attempt fails), unique IDs, unknown-schema-version fail-closed, full
contract validation, origin/status/seam vocabulary validation, follow-up ownership for
guard_missing + policy_blocked, unresolved-decision naming for policy_blocked,
no-runtime-claim proof for not_applicable, guard_verified only with a real seam,
buyer-language checks (numeric-confidence regex + the REAL BuyerCopySafety production
seam -- not a reimplementation), credential/customer-content hygiene, byte-stable
double serialization + SHA-256, machine-readable result written to the test output
location only. Seam tests (`ProtocolFailureSeamTests`, 8 C# tests + python
`test_protocol_failure_corpus_seams.py`, 3 tests) exercise ONLY real production seams:
SourceReadinessClassifier, SettlementProvenance, ArtifactDirectionConsistencyEvaluator,
ToolGateway (DI-resolved), DuplicateRunGuard, BuyerSettledRecapProjection, buyer DTO
contracts, BuyerCopySafety, python _parse_response, model_metering. No production
logic is duplicated in the harness; the fixture-only validator validates fixture
schema and never claims runtime proof.

## 3. determinism + independence proof

Corpus serialization double-run byte-equal (asserted); all fixture data static (no
clock reads, no random identifiers); harness runs offline -- no network, no model, no
database, no environment secrets; the only artifact written is
`protocol-failure-corpus-results.json` in the test output directory. Python seam
tests are pure-function calls.

## 4. test results (2026-07-15, dai branch @ f057a39)

| suite | command | result |
|---|---|---|
| focused C# corpus+seams | dotnet test DevCore.Api.Tests --filter FullyQualifiedName~ProtocolFailureCorpus | 27/27 passed, 0 failed, 0 skipped (~190ms) |
| focused python seams | pytest tests/test_protocol_failure_corpus_seams.py | 3/3 passed |
| FULL C# regression | dotnet test DevCore.Api.Tests | 1262/1262 passed (was 1235 at 85a8831; +27) |
| FULL python regression | pytest tests | 456/456 passed (was 453; +3) |
| vitest | NOT RUN -- no Angular file touched | n/a |

Pre-existing nullable-reference warnings in the suite unchanged; no new warning class
introduced. Passing totals are NOT semantic validation -- they prove the corpus
contract and the characterized seam behaviors only.

## 5. current vs target behavior (compact; full detail in the fixtures)

Fail-closed today: identity ambiguity/mismatch, settlement
residue 422 (settlement write path), direction-integrity 422, tool gateway,
duplicate guard, posture clamp (parse-time), buyer copy suppression, confidence-free
buyer contracts. Deterministic-but-not-final: readiness eligibility (fail-closed
CLASSIFICATION consumed procedurally -- the paid creation path never consults it);
provider absence collapsing to "missing" (no unavailable/stale states); honest recap
states without malformed-residue validation. Missing: cross-source contradiction
detection (PF-04). Policy-blocked: staleness handling (PF-02). Not applicable:
delivery/entitlement runtime (PF-15; procedural contract preserved).

## 6. runtime gaps discovered + follow-up ownership

No NEW gap outside PH-02..PH-06 was discovered; every guard_missing/policy_blocked/
characterized gap maps to an existing owner: PF-02 -> PH-02+PH-03; PF-03 -> PH-02 +
G-06 design; PF-04 -> PH-02+PH-03; PF-08 earlier-blocking -> PH-03; PF-09 execution-
blocking posture -> PH-03/R-04; PF-12 malformed-residue validation -> PH-04; PF-15 ->
PH-05. Deferred candidate notes: none required (nothing fell outside the six).
Release relevance: none of the gaps blocks RC Gate 1 (all are hardening-class).
Tenant/buyer relevance: PF-12/PF-13/PF-14 buyer-relevant (guards verified or
characterized honest); PF-10 tenant/authorization-relevant (verified).

## 7. RC-neutrality evidence

`git diff 85a8831 -- <all production trees>` on the branch = EMPTY. Changes = 3 new
files under DevCore.Api.Tests/ProtocolFailureCorpus + 1 new python test file; no
production source, config, schema, prompt, route, or startup/shutdown change; test
projects are not part of any production artifact; dai/main untouched at 85a8831;
csproj phantom untouched. Integration remains separately reviewed -- this
classification is the author's evidence, not integration authority.

## 8. limitations

Guard verification is at the SEAM level; end-to-end endpoint behavior relies on the
existing integration suites cited per fixture. PF-09 verifies visibility, not spend
blocking. PF-15 asserts a documented procedural contract, not code. The corpus does
not evaluate semantic quality of model prose (G-03's scope). No evidence class above
fixture-proven is claimed for the new tests.
