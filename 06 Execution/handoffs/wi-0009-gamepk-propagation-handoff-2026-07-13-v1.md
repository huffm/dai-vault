---
title: "WI-0009 Propagate gamePk Through CompetitionMatchupInput v1 -- slice handoff (2026-07-13)"
type: "handoff"
date: "2026-07-13"
status: "complete"
project: "DAI"
slice: "WI-0009 Propagate gamePk Through CompetitionMatchupInput v1"
repos:
  dai: "code (local branch, not pushed)"
  dai-vault: "docs-only (local, not pushed)"
tags:
  - execution
  - handoff
  - system-development
  - sports
  - identity
related:
  - "02 Platform/system-development/work-items/WI-0009-gamepk-propagation-through-competition-matchup-input.md"
  - "02 Platform/system-development/work-items/WI-0006-identity-safe-mlb-doubleheader-resolution.md"
---

# wi-0009 slice handoff -- gamepk propagation through competitionmatchupinput v1

**Disposition: IMPLEMENTATION COMPLETE, LOCAL ONLY. Review APPROVE. Nothing pushed.**

Governing WI: WI-0009 (status `complete`, disposition implementation-complete/local; MOC
registered). First slice both AUTHORIZED through the WI-0008 planner decision gate (12/12
principal-engineer conditions) and EXECUTED under the WI-0007 qualification gate.

## 1. what shipped

| file | change |
|---|---|
| `AgentRuns/AgentRunContracts.cs` | `CompetitionMatchupInput` gains optional `long? GamePk = null`, null-suppressed (`JsonIgnore` when null) |
| `AgentRuns/AgentRunService.cs` | passes `req.Input.GamePk` onto `SportsRunArtifact.RequestedGamePk` |
| `Controllers/AgentRunsController.cs` | Create rejects `gamePk <= 0` with 400 at the trust boundary -- before the pending row and before any spend |
| `AgentRuns/SportsRunArtifact.cs` | WI-0006 comment updated (superseded by this authorized contract expansion) |
| tests | `GamePkPropagationTests.cs` (7 new) + create-validation integration test (+1) |

**Contract:** absent field == null == exactly today's fail-closed path (WI-0006 unchanged);
present field carries exact event intent to the resolution seam the retriever already
consumes. Ordinary `InputJson` stays **byte-identical in both serializer profiles** (web AND
the default pascal-case profile the controller persists with); historical rows deserialize to
`GamePk = null`. No schema change, no migration, no new resolution logic.

## 2. verification

- Full `DevCore.Api.Tests`: **1127 / 0 / 0** (1120 -> +7). WI-0006's 20 doubleheader
  regressions untouched and green -- the fail-closed compatibility proof.
- Live, non-paid: `/health` 200; `/source-readiness` regression on the real 823357/823356
  doubleheader (ambiguity refusal + exact game 2, Chandler); create `gamePk=0` and negative
  -> 400 with **zero analyzer/model log lines**. Full paid generation deliberately NOT run;
  its identity path is proven deterministically through the same retriever seam
  (`explicit_game2_intent_reaches_retrieval_with_game2_identity`).
- Review: APPROVE, zero blockers. One review-grade catch folded in: the audit-trail test now
  mirrors the exact serializer profile production persists with, not just web options.
- Runtime cold start/end, independently verified (5/5 ports free, no processes, zero
  containers).

## 3. repo state

- before: dai `main` @ `88c9f09` == origin; vault `main` @ `05d5b10` == origin.
- after: dai `wi/0009-gamepk-propagation` 1 local commit (hash in current-slice/final
  report), **not pushed**; vault `main` 1 local commit, **not pushed**. Phantom + both
  intentional vault untracked files preserved.

## 4. what this does and does not change operationally

Doubleheaders are now safely CAPTURABLE by code -- but capture remains **not authorized**
(no-spend posture, operator-owned). Shipping this code does not reopen spend; an ambiguous
request without a gamePk still degrades to priors-only, unmatched.

## 5. deferred (not authorized, not started)

1. WI-0009 integration and push.
2. Doubleheader capture OPERATION (requires an explicit capture authorization).
3. First-class identity outcome statuses (WI-0006 deferral 2, independent).
4. WI-0002; WI-0003.

### Slice Synopsis

**Change:** The initiating generation request can carry an exact MLB event identity --
optional, null-suppressed `GamePk` propagated to the WI-0006 resolution seam, completed
locally.
**Reason:** Doubleheaders were safely rejected but never capturable; a real paid run was
already lost to this gap.
**Proof:** 1127/1127 tests (+7, incl. both-serializer-profile audit-trail proof); live
non-paid regression on the real 823357/823356 fixture with zero model-call log lines.
**State:** One local commit per repo, nothing pushed; WI-0009 implementation complete;
runtime cold.
**Next:** Separately authorize WI-0009 integration and push.
