---
title: "WI-0004 Truthful Platform API Shutdown v1 (platform api shutdown process match)"
type: "plan"
date: "2026-07-10"
status: "complete"
project: "DAI"
slice: "WI-0004 Truthful Platform API Shutdown v1"
repos:
  dai: "code"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - tooling
related:
  - "06 Execution/reports/nightly-closeout-2026-07-09-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "06 Execution/handoffs/wi-0005-starter-cache-handoff-2026-07-11-v1.md"
---

# WI-0004 truthful platform api shutdown v1

**Status: COMPLETE** (banner corrected forward 2026-07-24 completion audit; the stale
"IN PROGRESS" opener predated closure -- closure evidence is the links block and definition
of done; integrated to dai/main at `e8050a9`). Originally: Implementation AUTHORIZED
2026-07-11 (Truthful Platform API Shutdown v1
execution prompt). Registered 2026-07-10 BACKLOG from a follow-up in the nightly closeout;
unblocked after WI-0005 integrated. Filename kept stable to preserve the MOC wikilink; title
updated to the authorized scope.

## problem

`dai/scripts/stop-platform-api.ps1` prints `done` / `no platform api process found` and exits 0
without verifying that the platform API actually stopped or that its listener on port 5007 was
released. This is a **false success**: an operator (or a closeout slice) trusts exit 0 while the
API may still be bound, forcing manual PID cleanup.

## reproduction (this slice, 2026-07-11)

The 2026-07-09 closeout hypothesized the child `DevCore.Api.exe` survives when the `dotnet.exe`
host is killed. **That hypothesis did NOT reproduce**: under `dotnet run`, killing the
`dotnet.exe` host also brings down the `DevCore.Api.exe` child (job-object semantics) and 5007
freed. Not inventing a fix for a defect that does not reproduce.

The **actual, demonstrable** defect was reproduced faithfully:

- Started `DevCore.Api.exe` directly with `ASPNETCORE_URLS=http://localhost:5007` (a supported way
  to run the built app, and the shape of any orphaned apphost). Port 5007 owned by
  `DevCore.Api.exe` pid 28992; **zero** `dotnet.exe`-with-`DevCore.Api` processes.
- Ran the current `stop-platform-api.ps1` -> output `no platform api process found`, **exit 0**.
- Port 5007 **STILL LISTENING** pid 28992 afterward. **False success confirmed.**

## root cause

Two independent defects, either sufficient for a false success:

1. **Narrow target identity.** The script matches only `Name -eq "dotnet.exe" -and CommandLine
   -match "DevCore\.Api"`. It cannot see the `DevCore.Api.exe` apphost (the process that actually
   binds 5007 when launched directly or orphaned).
2. **No verification.** It prints `done`/`no ... found` and exits 0 unconditionally -- it never
   checks process exit or port release. Even in the case where the stop happened to work, success
   was asserted, not earned.

Secondary: assigns to `$matches` (a PowerShell automatic variable); `$ErrorActionPreference =
"SilentlyContinue"` swallows `Stop-Process` failures.

## intended outcome

The script succeeds only after independently verifying the stopped state. When the target cannot
be stopped within a bounded window, it returns non-zero with actionable diagnostics.

## success invariant

Shutdown is successful only when **port 5007 is no longer listening AND no platform-api host
process remains** (`DevCore.Api.exe`, or `dotnet.exe` running `DevCore.Api`). Already-stopped
satisfies the invariant and is a success.

## design (smallest correct)

- **Target identity** (strongest first): the process that owns port 5007
  (`Get-NetTCPConnection -State Listen -LocalPort $Port`) when it is a platform-api process; plus
  any platform-api processes found by metadata. A platform-api process is
  `Name -eq 'DevCore.Api.exe'` OR (`Name -eq 'dotnet.exe'` AND CommandLine matches `DevCore\.Api`).
  The launching `pwsh.exe` and the script's own PID are excluded. Never terminate all `dotnet`/
  `pwsh` processes by name.
- **Ambiguity guard:** if a NON-platform-api process owns 5007, do not kill it -- return non-zero
  with diagnostics (which pid/name owns the port).
- **Truthful completion:** after issuing stops, poll (bounded `TimeoutSeconds`, `PollMilliseconds`
  interval, re-reading fresh state each iteration to avoid PID reuse) until the success invariant
  holds or the timeout expires. Success output is emitted only after the invariant verifies.
- **Exit codes:** 0 stopped-or-already-stopped; 2 ambiguous port owner; 3 timeout / still bound.
- **Idempotent:** no target and port free -> success, "already stopped", no process mutation.
- **Testable:** pure decision functions (`Test-IsApiProcess`, `Resolve-ShutdownPlan`,
  `Test-ShutdownComplete`) dot-sourced and tested with in-memory fake process/port tables, using
  the `$MyInvocation.InvocationName -eq '.'` guard convention from check-settlement-finals.ps1.
  No real sleeps in unit tests.

## in scope

`stop-platform-api.ps1`; its helper functions; a deterministic PowerShell test harness; live
start/stop verification; process-exit + port-release verification; bounded polling; truthful exit
codes; idempotent already-stopped; concise diagnostics.

## out of scope

Rewriting `start-platform-api.ps1` (no change needed); general process supervisor; Windows-service
conversion; Docker orchestration; cross-platform process framework; FastAPI/Angular/DB-container
shutdown; WI-0005/sports/decision changes; doubleheader work; broad dev-script cleanup; push/merge/PR.

## risks (evaluated)

killing an unrelated process (mitigated: platform-api predicate + port-owner-must-be-api guard,
self/pwsh excluded); killing all dotnet (mitigated: name+commandline predicate, never name-only);
trusting a launcher PID (mitigated: verify port release + process exit, not the stop call);
orphaning a child (mitigated: both host and apphost are targets; verification catches survivors);
PID reuse (mitigated: re-read fresh each poll); exit/port race (mitigated: invariant requires BOTH
port-free AND no-api-process); fixed sleeps / infinite wait (mitigated: bounded poll, no long
sleep, pure logic unit-tested); premature success (mitigated: output after verification only);
nonzero on idempotent already-stopped (mitigated: AlreadyStopped -> exit 0); test cleanup leaving
processes (mitigated: unit tests spawn no processes; live scenarios clean by pid).

## acceptance criteria

1. Already-stopped (no target, port free) returns success, touches nothing.
2. Normal stop returns success only after port 5007 is free AND no api process remains.
3. A listener the old filter missed (DevCore.Api.exe with no dotnet parent) is stopped, or the run
   fails non-zero -- never a false success.
4. Stop failure / timeout returns non-zero with diagnostics (port, pid, listener, timeout).
5. A non-api process owning 5007 is not killed; the run fails non-zero with diagnostics.
6. Idempotent second invocation returns success ("already stopped").
7. Success output is emitted only after verification.
8. Unrelated processes are never terminated.
9. Deterministic unit tests pass; live scenarios A-D pass.
10. No locked-layer change; nothing pushed.

## test plan (written before implementation)

`dai/scripts/test-stop-platform-api.ps1` (new): dot-source the script, assert the pure functions
against fake snapshots -- IsApiProcess (DevCore.Api.exe true / dotnet+DevCore.Api true / other
dotnet false / pwsh false / unrelated false); Resolve-ShutdownPlan (already-stopped / stop-both /
ambiguous-port -> no target); Test-ShutdownComplete (listening-or-procs-remain false, both-clear
true). Live: Scenario A normal stop, B idempotent, C direct-exe listener the old script missed, D
unrelated-process safety.

## rollback

Single-script change plus one test file. Revert = the two commits. No application code, schema,
persisted state, or runtime contract outside the shutdown workflow.

## approval boundary

WI-0004 authorizes only truthful shutdown remediation and its verification. It does NOT authorize
paid calls, sports operations, starter-cache changes, decision-layer changes, doubleheader work,
or integration to remote branches (no push/merge/PR this slice).

## required lifecycle stages

All eight of [[implementation-lifecycle]]. Lite tier (bounded tooling fix, no contract/doctrine
surface); no review gate beyond code review.

## links (completed at close)

- work item: WI-0004
- branch: `wi/0004-truthful-api-shutdown` (from dai `4693b9d`) — pushed to origin 2026-07-11; retained
- pr: — (merged direct via fast-forward: `4693b9d..e8050a9`)
- commits: dai `e8050a9` (script + test) — **integrated to dai/main and pushed** 2026-07-11
  (clean fast-forward, no merge commit); dai/main == origin/main at `e8050a9`
- tests: `dai/scripts/test-stop-platform-api.ps1` (15 offline assertions, all pass) + live
  scenarios A-D (normal / idempotent / direct-exe blind spot / unrelated-process safety)
- verification notes: unit 15/15; reproduction proved false success (DevCore.Api.exe on 5007,
  old script exit 0 / port bound); live A-D pass; dai-code-reviewer APPROVE (0 blockers);
  handoff `06 Execution/handoffs/wi-0004-truthful-shutdown-handoff-2026-07-11-v1.md`
- docs updated: this WI; `06 Execution/handoffs/current-slice.md`; MOC (status complete)
- lessons: a shutdown is successful only when the process is gone and its listener released --
  never because a termination request returned

## reproduction correction (recorded)

The 07-09 "child survives when host is killed" hypothesis did NOT reproduce (dotnet run
job-objects the child; it dies with the host). The real, faithful reproduction was a
`DevCore.Api.exe` bound to 5007 with no dotnet parent (direct-exe / orphan shape): the old
script reported `no platform api process found` / exit 0 while 5007 stayed bound. The fix
addresses that proven defect (narrow filter + no verification), not the unreproduced hypothesis.

## definition of done

Per [[implementation-lifecycle]]: reproduction proven; success invariant explicit; narrow target
identity; verification-gated success; idempotent; non-zero on failure/timeout; unrelated processes
protected; unit tests + live scenarios pass; code review APPROVE; no locked-layer change; nothing
pushed.
