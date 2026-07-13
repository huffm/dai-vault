---
title: "WI-0009 Propagate gamePk Through CompetitionMatchupInput v1"
type: "plan"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0009 Propagate gamePk Through CompetitionMatchupInput v1"
repos:
  dai: "code"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - sports
  - identity
related:
  - "02 Platform/system-development/work-items/WI-0006-identity-safe-mlb-doubleheader-resolution.md"
  - "02 Platform/system-development/work-items/WI-0008-evidence-grounded-next-slice-planner.md"
  - "06 Execution/plans/platform-delivery-timeline-v1.md"
---

# WI-0009 propagate gamepk through competitionmatchupinput v1

**Slice type:** implementation. **Opened:** 2026-07-13. **Authorized:** operator decision
2026-07-13 accepting the WI-0008 planner recommendation (12/12 principal-engineer conditions).

## problem

The initiating generation request cannot express an exact MLB event identity.
`CompetitionMatchupInput` carries only `(competition, homeTeam, awayTeam, gameDate)`, and
`AgentRunService` constructs `SportsRunArtifact` without a `gamePk` -- so WI-0006's fail-closed
doubleheader behavior means ambiguous doubleheaders are safely REJECTED but never CAPTURABLE.
A real paid run (`6c9d433e`, MIL@PIT 823357) was already lost to this class.

## why now

WI-0006 built the identity-safe resolution path and WI-0008's planner ranked this first at
high confidence: it is the only candidate that unlocks a capability, it completes the WI-0006
seam (the retriever already consumes `SportsRunArtifact.RequestedGamePk`), and it needs no
paid calls to build or verify.

## intended outcome

An optional `gamePk` on the initiating contract propagates end-to-end so explicit Game 1 /
Game 2 requests generate under their exact event identities, while requests without it keep
today's fail-closed behavior byte-for-byte. Persisted `InputJson` for ordinary requests stays
byte-identical (null `gamePk` is not serialized).

## scope

- `AgentRunContracts.cs`: `CompetitionMatchupInput` gains `long? GamePk = null` with
  null-suppressed serialization.
- `AgentRunService.cs`: pass `req.Input.GamePk` into `SportsRunArtifact`.
- `AgentRunsController.cs` (Create): `gamePk <= 0` -> 400 BEFORE the pending row is written
  and before any model call.
- `SportsRunArtifact.cs`: update the WI-0006 comment now superseded by this authorized
  contract expansion.
- tests: contract serialization back/forward compatibility, propagation, validation,
  doubleheader generation identity (deterministic).
- vault: this WI, MOC, current-slice, handoff.

## out of scope

Capture authorization; paid calls; calibration; reconciliation; cross-sport identity
abstraction; status-model redesign; schema migration (none needed -- `InputJson` is an opaque
json column; additive optional field only); UI/Angular; WI-0002/0003; push/merge/PR.

## architecture / ownership boundary

The initiating request is the earliest layer that can express caller intent (WI-0006 decision
record). Event RESOLUTION stays owned by `MlbStarterClient`/`MlbEventResolver` (WI-0006);
this slice only carries intent to it. `/source-readiness` keeps its existing query-param seam
unchanged.

## key decisions

1. **Optional, null-suppressed.** `GamePk` defaults to null and is `[JsonIgnore]`d when null,
   so ordinary single-game `InputJson` serializes byte-identically to today -- no audit-trail
   drift, no read-model impact. Old persisted rows deserialize unchanged (missing property ->
   null) -- proven by test.
2. **Validation at the trust boundary.** `gamePk <= 0` is rejected 400 in Create before the
   pending AgentRun row is written and before any spend, mirroring `/source-readiness`.
3. **No new resolution logic.** Everything downstream of `SportsRunArtifact.RequestedGamePk`
   already exists and is regression-tested (WI-0006: exact-match validation, ambiguity
   refusal, v2 cache identity, mismatch fail-closed).

## risks

- persisted-contract drift -> mitigated: null suppression + old-row round-trip test;
- a bogus gamePk causing a wrong-game generation -> mitigated: WI-0006's identity-mismatch
  fail-closed path covers it (mismatched matchup -> no identity, priors-only) -- covered by a
  generation-path test;
- silent behavior change for existing callers -> mitigated: absent field == null == exactly
  today's path; full suite must stay green.

## acceptance criteria

1. Old `InputJson` (no gamePk) deserializes to `GamePk = null`; ordinary requests serialize
   without a `gamePk` property (byte-identical audit trail).
2. `GamePk` round-trips when present.
3. `AgentRunService` propagates `Input.GamePk` to `SportsRunArtifact.RequestedGamePk`.
4. Create rejects `gamePk <= 0` with 400 and writes no run row.
5. Deterministic doubleheader generation: explicit Game 1 pk -> Game 1 identity + starters;
   explicit Game 2 pk -> Game 2; no gamePk -> ambiguity path (priors-only, unmatched) exactly
   as before; mismatched pk -> fail-closed unresolved.
6. Full `DevCore.Api.Tests` green; builds clean; no locked-layer change; no migration.
7. Live (non-paid): `/health`; `/source-readiness` regression on the real doubleheader;
   Create-endpoint 400 validation (rejected before spend). Full paid generation is NOT run --
   its identity path is proven deterministically through the same retriever seam.

## verification plan

New `AgentRuns/GamePkPropagationTests.cs` (+ additions to existing suites where idiomatic);
`dotnet test DevCore.Api.Tests`; live scenarios above; guardrail greps.

## rollback / reversibility

Revert the commits: the field is additive and null-suppressed, so no persisted row, schema,
or historical data needs repair.

## approval boundary

WI-0009 authorizes this contract propagation only. It does not authorize capture, paid
execution, reconciliation, calibration, or any operational sports action. Local commits only;
push/integration separately gated.

## deferred work

- first-class identity outcome statuses (WI-0006 deferral 2, independent);
- doubleheader capture OPERATION (needs a capture authorization, not code).

## links

- work item: WI-0009 (ADO: AB#-- when wired)
- branch: `wi/0009-gamepk-propagation` (dai, at `d493f84`) -- **pushed 2026-07-13; retained**.
  Process note: the build slice accidentally committed directly to local dai/main (shell-cwd
  slip created the branch in dai-vault instead); local main contained exactly the one reviewed
  commit, so integration re-minted the convention branch at `d493f84` and pushed both. No
  rebase/amend; content identical to review.
- pr: -- (merged direct via fast-forward: `88c9f09..d493f84`)
- commits: dai `d493f84` (contract + service + controller + tests) -- **integrated to dai/main
  and pushed** 2026-07-13 (dai/main == origin/main at `d493f84`, main tree == branch tree);
  dai-vault `ba62cc8` (this WI + MOC + current-slice + handoff) + integration record, pushed
- tests: `AgentRuns/GamePkPropagationTests.cs` (7: old-row round-trip, null suppression in
  BOTH serializer profiles incl. the default profile the controller persists with, value
  round-trip, service propagation, null passthrough, exact-identity reach);
  `Integration/AgentRunsControllerTests.cs` (+1: create 400 before any row write). WI-0006's
  20 doubleheader regression tests untouched and green -- the fail-closed compatibility proof.
- verification notes: full `DevCore.Api.Tests` **1127 passed / 0 failed / 0 skipped** (1120 ->
  +7); build clean; live non-paid: /health 200, /source-readiness ambiguity + exact-game
  regression on the real 823357/823356 fixture, create gamePk=0 and negative -> 400 with zero
  analyzer/model log lines; runtime returned to cold and verified. Full paid generation NOT
  run (would spend); its identity path proven deterministically through the same retriever
  seam. dai-code-reviewer APPROVE, zero blockers.
- docs updated: this WI; MOC; current-slice; slice handoff
- lessons: a persisted-contract test must mirror the exact serializer profile production
  writes with -- a web-options assertion alone would have passed while the persisted
  (pascal-case) shape drifted.

## final disposition

Integrated and pushed (2026-07-13). Reviewed commit `d493f84` on dai/main == origin/main;
suite re-verified 1127/1127 immediately before integration; vault records pushed. Original
acceptance criteria preserved. Fully closed.
