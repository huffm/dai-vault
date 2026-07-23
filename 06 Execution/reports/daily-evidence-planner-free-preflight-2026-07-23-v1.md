---
title: "Daily Evidence Planner and Free Preflight 2026-07-23 v1"
type: "evidence-report"
date: "2026-07-23"
status: "complete -- fresh pass-1 built and planned; free preflight + zero-quota events gate ALL-QUALIFIED (5/5); paid screen GO recommended; stopped at operator boundary"
project: "DAI"
slice: "daily evidence operation 2026-07-23 (planner + free preflight phases; no work item)"
repos:
  dai: "unchanged (read-only; integrated CLIs invoked from main 85af96d)"
  dai-vault: "docs only; local branch ops/2026-07-23-daily-evidence, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - planner
  - market-contrast
related:
  - "06 Execution/patterns/daily-evidence-acquisition-operating-workflow-v1.md"
  - "06 Execution/reports/july-22-canary-settlement-reconciliation-2026-07-23-v1.md"
  - "06 Execution/reports/daily-evidence-planner-pass1-free-preflight-2026-07-22-v1.md"
---

# daily evidence planner and free preflight 2026-07-23 v1

## sequencing

July 22 settlement was reconciled FIRST (see the sibling reconciliation record), the
pooled calibration readout was refreshed from live rows, and only then was the July 23
planner run. The planner did not run against stale settlement state.

## live clocks

| moment | UTC | America/New_York |
|---|---|---|
| entry | 2026-07-23T14:55:18Z | 10:55:18 EDT |
| july-22 settlement written | 2026-07-23T15:04:21Z | 11:04:21 EDT |
| pass-1 as-of | 2026-07-23T15:08:01Z | 11:08:01 EDT |
| preflight completed | 2026-07-23T15:10:01Z | 11:10:01 EDT |
| events gate completed | 2026-07-23T15:10:19Z | 11:10:19 EDT |

## pass-1 (fresh request, current contracts)

Contract versions verified before use: CLI `daily-evidence-planner-cli/2.6`, request
`daily-evidence-planner-request/2.2`, board/planner `2.3`, envelope
`input-evidence-envelope/1.2`. Unlike July 22 (which replayed a frozen July-20 request),
today's request was built fresh from the live StatsAPI schedule because no frozen July 23
request existed. Policy verdict carried the same-morning refreshed readout: criterion
`discrimination_hybrid_v1`, `conclusions_allowed=false`, failing reasons
`discrimination_inverted` + `insufficient_market_disagreement`, `stale=false`.

Request validated `status: valid`; board published atomically. Outcome:
`EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE` (expected at pass-1 -- the market-contrast
envelope does not exist yet), evidence need NEEDED, cohort worthiness WORTHY, chosen
objective `evidence.market_disagreement`, primary limit 2, reserve limit 3.

Slate: 5 games, 5 distinct identities, no doubleheaders, all probable starters announced.

## free preflight (no paid call)

Terminal `completed_preflight_no_paid_call`, bundle schema
`market-contrast-screen-bundle/1.4` (binding-aware), 1 StatsAPI schedule call, 10 starter
attempts (0 failures), 1 db read, 0 db writes, 0 odds calls, $0.

## zero-quota events gate

One Odds `/events` call, quota audit PASSED (`x-requests-last=0`, used 287, remaining
213). 17-line result: 5 provider events joined; terminal
`qualified_binding_ready_for_separate_operator_decision`.

**Every candidate produced exactly one qualified provider-event binding** under
`provider-event-binding-policy/1.0` (bounded +/-60s window, eastern bracket
2026-07-23T04:00:00Z -> 2026-07-24T04:00:00Z). All five provider events carry the same
uniform +60s start offset that made July 22's exact-start-only join fail 1 < 4; the
integrated WI-0035 binding policy qualifies them deterministically (exact team pair, exact
orientation, one admissible event each, zero reversed-orientation, zero ambiguity).
Capacity rule: 5 qualified >= 4 -- the rule that returned NO-GO on July 22 passes today.

## candidate disposition (all times UTC; admission cutoff = start - 60 min)

| rank | gamePk | game | start | cutoff | binding (provider event / fingerprint) | disposition |
|---|---|---|---|---|---|---|
| 1 | 822785 | TB @ TOR (Seymour v Bieber) | 19:07 | 18:07 | `36ba7a8a8c46e8cc308c1dd037995889` / `ea727fa3...` | PRIMARY -- widest safe window, fully qualified |
| 2 | 823042 | ARI @ STL (Pfaadt v McGreevy) | 21:15 | 20:15 | `8e2411581a27c6dccd9eae0b233fe64c` / `6c30c0e3...` | PRIMARY -- fully qualified |
| 3 | 824247 | KC @ DET (Dobnak v Melton) | 22:40 | 21:40 | `38d3052ec454022a14314b324e0ff5f3` / `46fa962b...` | RESERVE -- fully qualified |
| 4 | 824406 | MIN @ CLE (Bradley v Williams) | 17:10 | 16:10 | `5c7e77b42c0db49ce882d3c9473d74c1` / `8c57f975...` | RESERVE (time-boxed) -- qualified but cutoff ~16:10Z leaves a narrow decision window |
| 5 | 824893 | SD @ ATL (Hart v Sale) | 16:15 | 15:15 | `22fc220be6958e93fba4354054d8fd16` / `d9df7a2f...` | OPERATIONALLY INELIGIBLE -- cutoff passed during the operation; excluded by lead-time rule, not by identity |

Primary/reserve split follows the standing limits (primary 2, reserve 3). Wildcard math:
a 2-run scheduled flight admits floor(2/4) = 0 wildcards.

## slate classification: PAID_SCREEN_GO_RECOMMENDED

At least one candidate (in fact four live ones) passed every free gate. The evidence
posture (readout: manual paid capture allowed; tuning and buyer claims blocked) supports a
controlled paid screen. The next paid operation is ONE `/odds` screen call (h2h+spreads,
us region, eastern bracket) at an expected quota cost of 2 credits covering all qualified
candidates, followed by the deterministic pass-2 replay and, only under existing
thresholds and a separate capture authorization, up to 2 capture runs (~3 further credits
+ ~$0.001 model cost each, ceiling $0.01).

Latest safe operator decision times: ~15:50Z to still include MIN@CLE (rank 4); ~17:45Z
for the rank-1 TB@TOR window; ~19:55Z for rank-2 ARI@STL alone. Nothing has been screened,
reserved, captured, or promised: authority ledgers are all-false everywhere and this
record authorizes nothing.

## call ledger (whole 2026-07-23 operation, all phases)

| call | count | cost class |
|---|---|---|
| statsapi schedule (finals gate, slate source, preflight) | 3 | free |
| statsapi starter/pitcher stats | 10 | free |
| statsapi linescore verification | 1 | free |
| odds `/events` | 1 | zero-quota (audited, last=0) |
| odds `/odds` | **0** | paid -- NOT called |
| platform api reads (precheck, rows x2, metrics x2, evaluation) | 6 | free local |
| platform api settlement write (`/reconcile`) | 1 | free local (authorized) |
| model calls | **0** | -- |
| migrations applied | 0 | -- |
| paid cost | **$0** | -- |

## artifacts (outside both repositories)

`<DAI_WORKSPACE_ROOT>/daily-evidence-flight-2026-07-23/attempt-1/` and
`<DAI_WORKSPACE_ROOT>/events-gate-2026-07-23/attempt-1/`:

| file | sha-256 |
|---|---|
| `pass1-request.json` | `3c3cda8da651ff92d03da1044228f942ed5b2f20081b0160afd18abd2968543a` |
| `planner-pass1-board.json` | `58f7a54f4d9f2a4a161a165588153f13fefc63e2df890852ecfc3cccc09aef46` |
| `preflight-bundle.json` | `623eb88ae33f4375ae606764f93c4783756a3ab39934f9f8a7f27183e151b834` |
| `events-gate-artifact.json` | `73e7384b9b749f89d917854159a99c4ce16ea4fa285c6875caca53c20cafb374` |
| `slate.json` | `58e124b4ddfd1a153f5de9e3cca1dfd4f3ffc057751bbf7d9fb504dfb9afe4f0` |
| `statsapi-slate-source.json` | `ff0ffdac3de2e2e04ef2d43d1b139cbb9f75f03921a2469dc4f6048537f9c4ab` |

Plus attempt manifests and stdout/stderr logs (all stderr empty). No machine path,
secret, API key, or raw provider payload is persisted in either repository.

## what was not done

No paid `/odds` call, no quota reservation, no pass-2 board, no flight plan or freeze, no
capture, no buyer run, no settlement of any unplayed game, no source or schema change, no
merge, no push, no remote branch, no PR.
