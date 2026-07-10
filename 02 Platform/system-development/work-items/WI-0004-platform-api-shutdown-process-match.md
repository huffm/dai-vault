---
title: "WI-0004 Platform API Shutdown Process Match"
type: "plan"
date: "2026-07-10"
status: "blocked"
project: "DAI"
slice: "WI-0004 platform api shutdown process match"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - system-development
  - work-item
  - backlog
  - tooling
related:
  - "06 Execution/reports/nightly-closeout-2026-07-09-v1.md"
  - "06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
---

# WI-0004 platform api shutdown process match

**Status: BACKLOG. Implementation authorization: NOT AUTHORIZED.**
Registered 2026-07-10 from a follow-up recorded by the 2026-07-09 nightly closeout and
reproduced with live evidence during the 2026-07-10 operational slice. This spec records
the defect and the decision space; it does not pre-decide the fix.

## problem statement

`dai/scripts/stop-platform-api.ps1` selects processes with
`$_.Name -eq "dotnet.exe" -and $_.CommandLine -match "DevCore\.Api"`. Under `dotnet run`
the API is hosted by a `dotnet.exe` parent that launches `DevCore.Api.exe` as the child
that actually binds port 5007. The filter matches only the parent. The script therefore
kills the host, prints `done`, and exits 0 while the listener survives.

This is a **false success**, not merely a missed match: the operator gets an exit-0
"stopped" signal from a script that left the port bound. The 2026-07-09 closeout recorded
the symptom ("DevCore.Api ran as DevCore.Api.exe and ESCAPED stop-platform-api.ps1's
CommandLine match") but attributed it to the name alone.

## current evidence (reproduced 2026-07-10, read-only dry run, no process stopped)

Live during the day-1 settlement slice, with the API started by `scripts/start-platform-api.ps1`:

- Listener on 5007: `pid=27608 Name=DevCore.Api.exe`, CommandLine
  `...\DevCore.Api\bin\Debug\net10.0\DevCore.Api.exe`
- Host process: `pid=18988 Name=dotnet.exe`, CommandLine matches `DevCore\.Api`
- The script's exact filter, evaluated read-only, returned **only pid 18988**.
  Killing that alone leaves pid 27608 holding 5007.

Secondary latent defect, same file: line 3 assigns to `$matches`, a PowerShell **automatic
variable** that `-match` inside the `Where-Object` scriptblock also writes. The 2026-07-07
settlement-readiness slice recorded this exact class of bug (`$Home`/`$args`/`$matches`
collisions) as a standing PowerShell gotcha. It is not the cause of the escape but sits in
the same seven lines.

`$ErrorActionPreference = "SilentlyContinue"` (line 1) additionally suppresses the
`Stop-Process` failure that an elevated worker would raise — the 07-09 closeout needed an
unsandboxed `Stop-Process` for the elevated uvicorn worker, so silent failure is reachable.

## affected surfaces

- `dai/scripts/stop-platform-api.ps1` (sole implementation)
- Any slice-closeout procedure that trusts its exit code as shutdown proof
- `06 Execution/patterns/settlement-readiness-discipline-v1.md` (the "blocked runs are a
  console result" doctrine assumes scripts report honestly)

## classification

**Non-blocking defect.** It does not block settlement, capture, calibration, or any write
path. It degrades only closeout confidence, and only when the operator trusts exit 0
instead of verifying the port. Operational slices can and do verify ports directly.

## scope

Correct the process selection and make failure loud. Nothing else in `scripts/`.

## non-goals

No change to `start-platform-api.ps1`; no service-manager/supervisor introduction; no
change to agent-service or SQL shutdown; no change to any settlement, capture, calibration,
or runtime semantics; no new dependency.

## execution authority

None. This item is registered, not authorized. It must not be implemented inside an
operational cadence slice. It gets its own approved slice.

## activation gate

Implement when either: (a) a closeout is materially harmed by a false-success shutdown
(port left bound into a later slice, causing a start conflict), or (b) any slice already
opens `scripts/` for another approved reason and can absorb a bounded fix under TDD.

## decision space (all open)

1. Match on `CommandLine -match "DevCore\.Api"` regardless of `Name` (catches both host and
   child; must then avoid killing the pwsh wrapper that also matches).
2. Resolve the owner of port 5007 (`Get-NetTCPConnection -LocalPort 5007`) and stop that
   process plus its parent — the port is the thing the operator actually cares about.
3. Keep the name filter but add `DevCore.Api.exe` and verify the port is released before
   exiting 0.

Option 2 is the only one that makes the exit code mean "5007 is free". Note the live dry
run showed a third matching process (`pwsh.exe`, the wrapper that launched the script) —
any CommandLine-only widening must exclude it or the script will kill its own shell.

## acceptance criteria (for the future implementation slice)

1. After the script exits 0, no process listens on 5007 — asserted, not assumed.
2. When it cannot stop the listener, it exits non-zero and says which pid survived.
3. It never terminates its own host shell, or any process it did not identify as the API.
4. No automatic variables (`$matches`, `$args`, `$Home`, `$input`) are assigned.
5. Offline-testable selection logic (a pure function over a process list), mirroring the
   `check-settlement-finals.ps1` dot-source-testable pattern.

## test plan (written before implementation)

Mirror `dai/scripts/dev/sports/test-check-settlement-finals.ps1`: dot-source the selection
function and assert against injected process-table fixtures —
(a) host-only `dotnet.exe`; (b) host + `DevCore.Api.exe` child; (c) child only;
(d) an unrelated `dotnet.exe`; (e) the launching `pwsh.exe` present and never selected.
Plus one live integration check: start the API, run the script, assert 5007 free.

## verification commands

`pwsh -File dai/scripts/test-stop-platform-api.ps1` (new), then a live start/stop round trip
asserting `Get-NetTCPConnection -LocalPort 5007` returns nothing.

## dependencies

None. Independent of WI-0001/0002/0003.

## risks

- A CommandLine-only widening kills the operator's own `pwsh` shell (observed: pid 21244
  matched). Any fix must exclude the launching process explicitly.
- Over-broad matching in a shared dev box could stop an unrelated `dotnet.exe`.

## rollback / containment posture

Single-file script change; revert = one commit revert. No data, contract, runtime, or
buyer surface. Zero blast radius outside developer convenience.

## required lifecycle stages

All eight of [[implementation-lifecycle]]. Lite tier: this is a bounded tooling fix, not a
feature-class item; no contract or doctrine surface, so no spec review gate beyond owner
approval to authorize.

## required links (at close)

The standard 8 links per [[work-item-traceability]].

## definition of done

Per [[implementation-lifecycle]], plus acceptance criterion 1 demonstrated live (port free
after exit 0) and criterion 5's offline tests green.

## provenance

Symptom first recorded: `06 Execution/reports/nightly-closeout-2026-07-09-v1.md`.
Root cause reproduced with live pids: the 2026-07-10 day-1 settlement / day-2 capture
operational slice (see [[v2-day1-settlement-day2-capture-slice-2026-07-10-v1]]), which
verified the defect read-only and deliberately did **not** fix it — shutdown there was
completed by stopping the listener pid directly.
