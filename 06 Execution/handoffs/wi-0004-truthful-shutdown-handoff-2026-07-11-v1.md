---
title: "Handoff -- WI-0004 Truthful Platform API Shutdown v1 (2026-07-11)"
type: "handoff"
date: "2026-07-11"
status: "complete"
project: "DAI"
slice: "WI-0004 Truthful Platform API Shutdown v1"
repos:
  dai: "code (branch, local, not pushed)"
  dai-vault: "docs-only"
tags:
  - handoff
  - tooling
  - work-item
related:
  - "02 Platform/system-development/work-items/WI-0004-platform-api-shutdown-process-match.md"
  - "06 Execution/handoffs/wi-0005-starter-cache-handoff-2026-07-11-v1.md"
---

# handoff -- WI-0004 truthful platform api shutdown v1

## 1. disposition

COMPLETE. Reproduced, fixed, unit-tested, live-verified (A-D), reviewed, documented, committed
**locally on a branch**. Nothing pushed, no PR.

## 2. skills used

dai-skill-router (gate); dai-slice-runner (lifecycle); systematic-debugging (reproduction);
superpowers:test-driven-development (pure-function tests); dai-test-discipline (unit harness +
live ladder); dai-code-reviewer (APPROVE); dai-grill-with-vault (script vs doctrine);
superpowers:verification-before-completion; dai-docs-architect; dai-agent-handoff; dai-token-tight.
Not selected: none material. PS test-harness convention taken from test-check-settlement-finals.ps1.

## 3. starting state

dai main 4693b9d == origin (csproj phantom only); dai-vault b9d76d5 == origin (2 untracked).
Runtime cold. Branch `wi/0004-truthful-api-shutdown` cut from 4693b9d.

## 4. work item

`WI-0004-platform-api-shutdown-process-match.md` (filename kept for the MOC wikilink; title ->
"Truthful Platform API Shutdown v1"). BACKLOG -> in-progress (before code) -> complete.

## 5. baseline and reproduction

- start path: `start-platform-api.ps1` -> `dotnet run` -> dotnet.exe host + DevCore.Api.exe apphost
  (the apphost binds 5007 via launchSettings applicationUrl).
- The 07-09 hypothesis (child survives host kill) did NOT reproduce: killing dotnet.exe also
  dropped DevCore.Api.exe and freed 5007 (job-object semantics). Not fixed a non-existent defect.
- Faithful reproduction: `DevCore.Api.exe` started directly with
  `ASPNETCORE_URLS=http://localhost:5007` -> listener on 5007 with no dotnet parent. Current script
  -> `no platform api process found`, exit 0, **port 5007 still listening**. False success confirmed.

## 6. root cause

(1) narrow target filter (dotnet.exe+DevCore.Api only) cannot see the DevCore.Api.exe apphost;
(2) no verification -- success printed unconditionally. Secondary: `$matches` automatic-variable
assignment; `$ErrorActionPreference="SilentlyContinue"` swallowed Stop-Process failures.

## 7. architecture decision

Success invariant: **port 5007 released AND no api host process remains.** Target identity: the
port owner when it is an api process, plus api processes by name+commandline (`DevCore.Api.exe` OR
`dotnet.exe`+`DevCore.Api`), self + pwsh excluded, never name-only. Ambiguity guard: a non-api port
owner is never killed (exit 2). Truthful completion: bounded poll (TimeoutSeconds/PollMilliseconds,
fresh state each iteration -> pid-reuse safe), success emitted only after the invariant verifies.
Exit codes 0/2/3. Idempotent already-stopped -> exit 0. Smallest correct: one script + one test,
reuses Get-NetTCPConnection/Get-CimInstance/Stop-Process, no new dependency; pure decision
functions behind the dot-source guard for offline testability.

## 8. implementation

- `scripts/stop-platform-api.ps1` rewritten: Test-IsApiProcess / Resolve-ShutdownPlan /
  Test-ShutdownComplete (pure) + Get-PortOwnerPid / Get-ApiProcesses (i/o) + verified main.
- `scripts/test-stop-platform-api.ps1` new: 15 offline assertions (dot-sourced, fakes, no i/o).

## 9. deterministic verification

`pwsh scripts/test-stop-platform-api.ps1` -> **15 passed / 0 failed**, exit 0. No sleeps, no
processes, no ports -> nothing to clean up. Change is scripts-only; no compiled project touched,
so no .NET build/suite re-run was required for correctness (diff proves scope).

## 10. live verification

- A normal (dotnet run): before owner DevCore.Api.exe; script stopped the dotnet host, apphost
  "already gone", verified port free + no procs, exit 0.
- B idempotent: exit 0 "already stopped", nothing touched.
- C direct-exe blind spot: DevCore.Api.exe on 5007, no dotnet parent; script stopped it, verified
  port free, exit 0 -- the reproduced false-success is fixed.
- D unrelated-process safety: a foreign pwsh TcpListener on 5007 -> exit 2 "blocked", process
  survived (not killed); dummy then cleaned up by pid.
- Not exercised live: the exit-3 timeout path (forcing a truly un-killable listener safely is
  impractical); its predicate Test-ShutdownComplete is unit-covered. Informational, not a blocker.

## 11. code review

dai-code-reviewer APPROVE, 0 blockers. Checked: false-success paths, premature success output,
ambiguous targeting, broad killing, pid reuse, launcher-vs-listener, orphan child, port-release
race, timeout, idempotency, unrelated safety, flaky tests, cleanup, overengineering, unrelated
changes. One informational note (exit-3 not live-exercised).

## 12. guardrail proof

0 paid calls / 0 sports captures / 0 reconciliation / 0 outcome / 0 exclusion writes / 0
calibration mutation / 0 db mutation beyond ordinary local service startup. No prompt / model /
confidence / scoring / regime / Gate-4 / buyer / schema / Angular change. WI-0005 (MlbStarterClient)
untouched; WI-0002/0003 untouched; doubleheader work not started. DevCore.Data.csproj phantom
untouched; vault untracked files untouched. No force-push / history rewrite (nothing pushed).

## 13. commits and final state

dai `e8050a9` on branch `wi/0004-truthful-api-shutdown` (from 4693b9d). dai-vault docs commit on
main. **Both LOCAL, nothing pushed.** dai main unchanged at 4693b9d. Runtime returned to cold
(ports free, 0 containers, devcore-sql stopped) -- verified directly, not from a script exit code.

## 14. deferred work (listed, not authorized, not started)

- WI-0004 integration and push (merge `wi/0004-truthful-api-shutdown` to main + push) -- awaits
  separate approval.
- Candidate doubleheader gamePk disambiguation through the /source-readiness caller path.
- WI-0002 / WI-0003 remain BACKLOG. No new slice authorized here.
