---
title: "Market Contrast Events-Gate Observation 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "observation complete (exact_match_ready_for_separate_operator_decision); no paid screen authorized"
project: "DAI"
slice: "WI-0035 July 22 zero-quota events-gate observation"
repos:
  dai: "unchanged (read-only; operator invoked from integrated main)"
  dai-vault: "docs only; local branch wi/0035-events-gate-observation-2026-07-22, NOT pushed / NOT merged"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - observation
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "06 Execution/reports/market-contrast-live-screen-2026-07-22-v1.md"
  - "06 Execution/reports/market-contrast-events-gate-slice-3-2026-07-20-v1.md"
---

# market contrast events-gate observation 2026-07-22 v1

## purpose

Record the single authorized zero-quota cross-provider identity observation for the MLB
slate of 2026-07-22: one refreshed free preflight followed by exactly one Odds
`/v4/sports/baseball_mlb/events` request. The observation answers one question only --
**can the frozen authorized candidates be joined to provider events by exact identity and
start instant?** It authorizes no paid screen, no capture, and no runtime action.

## execution window (live clocks, not the architecture timestamp)

- authorization not-before: `2026-07-22T12:00:00Z` (08:00 EDT).
- live clock at gate entry: `2026-07-22T12:22:49Z` / 08:22:49 EDT.
- preflight: started `2026-07-22T12:36:58Z`, completed `2026-07-22T12:37:01Z`.
- `/events`: started and completed `2026-07-22T12:38:37Z` (bundle age at call ~96s, well
  inside the five-minute freshness limit).
- absolute ceiling `2026-07-22T16:35:00Z` was never approached (~237 minutes of margin).

## frozen inputs (read-only, verified by hash)

- `screen-2026-07-22/slate.json` SHA-256
  `37EE908B353EC241D5AB0AE4C5605D1C86278881FFDAE092C5DF296AE6E8651F` -- matches the frozen
  authorization.
- `screen-2026-07-22/pass1-request.json` SHA-256
  `0FCCAFB19D60D193BF4A10C9B461A70CBEF9F2FF16BFC4B5A8F1E13565886899` -- matches.
- target date `2026-07-22`; exactly **15** authorized gamePks, all distinct; no candidate
  was added, substituted, or hand-composed.

Attempt uniqueness: no `/events` attempt existed for this date. The pre-existing
`screen-2026-07-22/` call ledger records **one `/v4/sports/baseball_mlb/odds` request made
on 2026-07-20** under the earlier paid-screen authorization -- a different endpoint under a
different authorization, and not a prior events-gate attempt.

## observed provider facts

- provider events returned for the eastern date bracket
  (`2026-07-22T04:00:00Z` -> `2026-07-23T04:00:00Z`): **17**.
- HTTP attempts: **1**; retries: **0**; timeout budget 20s.
- Odds `/odds` requests: **0** (the events-gate code path cannot reach `/odds`).

### quota audit -- PASSED

- `x-requests-last`: `"0"` (exactly numeric zero -> zero-quota confirmed)
- `x-requests-used`: `"282"`
- `x-requests-remaining`: `"218"`
- `zero_quota_observed`: `true`; `audit_status`: `passed`

Used/remaining are unchanged from the 2026-07-20 paid screen, independently corroborating
that this observation consumed no credits. No credential, secret-bearing header, or raw
provider payload was persisted in either repository.

## deterministic join result (integrated predicate, never broadened)

| fact | value |
|---|---|
| candidates | 15 |
| screenable | 13 |
| skipped (preblocked) | 2 |
| unresolved | 0 |
| same-orientation team pairs | 13 |
| reversed orientation | 0 |
| **exact matches** | **1** |
| start-instant mismatches | 12 |

- exact match: gamePk **823438**, `los-angeles-dodgers@philadelphia-phillies`, scheduled
  `2026-07-22T22:40:00Z`, provider event `111a955795876d50988b15c219ce0796`, start delta
  **0s**.
- skipped: gamePk **823518** (`caller_start_mismatch`), gamePk **824732**
  (`starters_not_announced`).
- start-delta distribution across the 12 mismatches: **+60s x 11**, **-240s x 1**.

## interpretation -- separated from observed fact

**Observed (fact):** team identity and orientation resolve cleanly. All 13 screenable
candidates produced a same-orientation team-pair hit and zero reversed-orientation hits, so
alias/orientation handling is not the limiting factor. The exact-match predicate failed for
12 of 13 candidates purely on the start instant.

**Inference (not yet proven):** the +60s offset on 11 of 12 mismatches looks systematic
rather than random -- consistent with the two providers publishing the same scheduled first
pitch with different second-level rounding. The single -240s case is distinct and may be a
genuine schedule revision.

**Unknown:** whether the +60s offset is stable across dates, and whether it is a provider
convention or an artifact of this slate. One observation cannot establish that.

This report deliberately does not convert the inference into a tolerance change. Widening
the exact-match predicate is a correctness decision about identity joins and requires its
own authorization and fixture-backed evidence.

## terminal outcome

Operator terminal status, used without reinterpretation:

`exact_match_ready_for_separate_operator_decision`

The dated observation is **complete**: a closed observed verdict with a passing zero-quota
audit. Per the operator contract, a later bounded `/odds` screen **may be proposed but is
not authorized by this observation**.

## ledger -- what ran and what did not

Ran: one StatsAPI schedule/hydrate batch (1 schedule request, 22 bounded starter attempts,
0 failures); one tenant-scoped active-run database **read**; one `/events` request.

Database writes: **0**. Odds `/odds`: **0**. Model calls: **0**. Tool Gateway: **0**.
AgentRun creation: **0**. Capture, screening spend, scheduling, settlement, reconciliation,
tuning: **0**. Paid cost: **$0**.

Every authority field in both the preflight bundle (8 fields) and the events artifact
(8 fields) is `false`.

## schema compatibility under the unapplied migration

The WI-0036 Slice-3 migration `20260722100648_AddAgentRunFlightSelectionWriteback` remains
**generated but unapplied**. The narrow active-run read was re-proven compatible before the
source call and then observed live; the executed SQL was:

```sql
SELECT [a].[ExternalGameId] AS [Key], COUNT(*) AS [Count]
FROM [AgentRuns] AS [a]
WHERE [a].[TenantKey] = @tenantKey AND [a].[SourceProvider] = N'mlb_statsapi'
  AND [a].[ExternalGameId] IS NOT NULL AND [a].[ExternalGameId] IN (...)
  AND [a].[ExclusionReason] IS NULL
GROUP BY [a].[ExternalGameId]
```

It selects only the grouped key and count and filters on four pre-existing columns. None of
the nine new flight-selection columns is referenced, which is why the read succeeds against
the unmigrated schema. No migration was applied and no database write occurred.

## external artifacts (outside both repositories)

`<DAI_WORKSPACE_ROOT>/events-gate-2026-07-22/attempt-1/`

| file | sha-256 |
|---|---|
| `preflight-bundle.json` | `84c8f7e6a487359d7b0afec036a4484a8915688c9c253ad5cb50632e94fecfd0` |
| `events-gate-artifact.json` | `26cdf20ce1a4b362929ee343134fe314ebcc0da9aca0062a69879d20ae554415` |
| `preflight-stdout.log` | `75f418970d6b8fd1046f088b0faaebf484e0c5c5bc1c03c81dbab148983de100` |
| `events-stdout.log` | `c7d1b8ccaae5bc343bb3b07e202acca5417b62d679ce78ed39ab05b57464be2a` |

Both stderr logs are empty (`e3b0c442...`). No recovery artifact was produced; publication
succeeded on the first attempt. Destination was claimed exclusively before the source call
and nothing pre-existing was deleted or overwritten.

Operator/contract versions: bundle `market-contrast-screen-bundle/1.3`, events artifact
`market-contrast-events-gate/1.0`, operator `market-contrast-events-gate-operator/1.1`,
adapter `market-contrast-source/1.1`, attempt id `7cebf9a1249a`.

## what this does not authorize

No paid `/odds` screen, capture, AgentRun, model call, scheduling, settlement,
reconciliation, migration application, runtime activation, or commercial action. WI-0036
Slices 1-3 remain integrated and unchanged by this observation; Slices 4-6 remain deferred.
Any paid wildcard flight remains separately authorized.

## recommended next action (proposal only)

An offline, fixture-backed investigation of the start-instant offset: characterize whether
the +60s delta is systematic across dates using already-captured artifacts, and decide
deliberately whether the exact-match predicate should admit a bounded start tolerance. That
decision changes identity-join correctness and therefore needs its own authorization and
regression fixtures. No provider call is required to begin it.
