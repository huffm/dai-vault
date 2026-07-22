---
title: "Provider-Event Binding Seam Audit 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "audit complete with RED evidence; full vertical NOT implemented -- fail-closed, decomposition proposed"
project: "DAI"
slice: "WI-0035 / WI-0036 provider-event binding vertical (audit phase only)"
repos:
  dai: "test-only commit on local branch wi/0035-provider-event-binding; NO production code changed; NOT pushed"
  dai-vault: "docs only on local branch wi/0035-provider-event-binding; NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - audit
related:
  - "06 Execution/reports/market-contrast-start-instant-normalization-analysis-2026-07-22-v1.md"
  - "06 Execution/reports/exact-identity-core-canary-capture-2026-07-22-v1.md"
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
---

# provider-event binding seam audit 2026-07-22 v1

## outcome

The required gap audit is **complete and executably proven**. The full binding vertical is
**not implemented**; this slice stops fail-closed with the audit and RED evidence committed
locally, plus a decomposition for the next passes.

This is a deliberate stop, not an abandonment. A partially propagated binding is worse than
none: it would create the appearance of a verified provider identity at the trust boundary
while execution still fell back to team/date matching, and a half-completed contract-version
cascade would break the cross-runtime vectors that currently keep the two runtimes honest.

## the corrected framing (documentation correction)

Preserved as history: the local start-instant analysis concluded that a **universal
scheduled-time rounding function is refuted**. That conclusion stands and is not reopened --
two scheduled instants each mapped to two different provider deltas (`22:40:00Z` -> `+0s`
and `+60s`; `00:10:00Z` -> `+60s` and `-240s`).

**Correction:** refuting a global transform does **not** refute a bounded, unique,
per-event identity join. Those are different claims. A safe join needs exact teams and
orientation, a narrow evidence-backed window, exactly one admissible provider event, a
producer-derived event ID, and propagation of that ID into execution -- not a rounding
function.

Supporting facts already captured: the second independent observation ~104 minutes after
the first showed **each event's delta stable to the second**, and the separately authorized
one-run core canary corroborated the exact unique case for gamePk 823438 (provider event
`111a955795876d50988b15c219ce0796`). The canary did **not** exercise screen-to-execution
propagation -- it succeeded under an explicit one-team-pair restriction -- and remains
non-wildcard, non-settled, pre-binding technical evidence. The `-240s` case remains excluded.

## proven seam gaps (RED, executable)

`dai` commit `96b935fd3dd329af59a1add757fc100ec967a3b5` adds
`DevCore.Api.Tests/Sports/ProviderEventBindingGapTests.cs` -- four characterization tests
that assert what the code does **today**, so the suite stays green while the defect is
proven. No production code was changed.

| gap | proven behavior | source |
|---|---|---|
| **1. doubleheader mis-binding** | with no provider event id, retrieval returns the **first listed** event for the team pair regardless of start instant. For a BAL@BOS doubleheader (`17:36Z`, `23:10Z`) a run intended for game 2 silently receives game 1's market. | `OddsMarketClient.FetchSpreadAsync` lines 313-319 |
| **1b. reversed orientation binds** | the fallback predicate accepts `(home==away && away==home)`, so a reversed listing still binds -- which the screening side forbids outright. | same predicate |
| **2. no re-verification by id** | the by-event-id path selects the exact id but never re-checks the returned event's home/away/orientation/date-bracket/start-delta against any frozen binding. There is no parameter through which an expected binding could be supplied. | `GetBaseballSpreadByEventIdAsync` -> `FetchSpreadAsync(matchEventId:)` |
| **3. contract cannot carry a binding** | `MarketSpreadInput` is exactly `(Competition, HomeTeam, AwayTeam, GameDate)`. `SportsRetriever` builds it from team names, so no frozen binding can reach the Tool Gateway or the client. | `Tools/Handlers/MarketSpreadHandlers.cs:9`; `SportsRetriever.cs:136-139` |

A fourth structural fact, from the screening side: `MarketJoinDiagnostics.Analyze` binds
**only** on exact UTC start equality (`e.CommenceTime == candidateStartUtc`). The eleven
stable `+60s` candidates therefore cannot bind at all today, which is why only one of
thirteen screenable candidates ever joined.

## implementation map (for the next passes)

Current path and where the provider event ID lives or dies:

```text
StatsAPI candidate            identity authoritative (gamePk, refs, start)
  -> MarketJoinDiagnostics    computes delta; binds ONLY at 0s; emits OddsEventId
  -> screen/events bundle     PRESERVES odds_event_id per candidate
  -> input-evidence envelope  DROPS it (normalized_result carries classification only)
  -> Planner Pass 2 board     DROPS it (candidates carry identity + classification)
  -> WI-0036 flight plan      DROPS it (no market-binding member)
  -> flight-selection prov.   DROPS it (no provider event member)
  -> controller boundary      cannot validate what is not carried
  -> SportsRetriever          rebuilds from TEAM NAMES ONLY
  -> Tool Gateway input       MarketSpreadInput has no binding member
  -> OddsMarketClient         FirstOrDefault, either orientation  <-- unsafe
  -> MarketSnapshot           records whichever event was selected
```

The binding is produced correctly at the screen and then **dropped at the envelope**, four
layers before execution. Every downstream layer therefore re-derives identity from team
names. That is the root shape of the defect: it is not a matching-tolerance problem, it is a
**propagation** problem.

No pre-existing canonical binding authority closes this seam, so building one is not
duplicative.

## why this slice stopped here

The remaining work spans two runtimes and a versioned-contract cascade:

1. a canonical pure binding authority with the 60s window in **one** versioned policy;
2. reuse by both the free events gate and the paid source adapter (bundle + artifact bumps);
3. carrying the binding through envelope -> planner request/board/CLI -> flight
   request/plan/CLI -> flight-selection provenance (each a versioned wire change);
4. regenerating **all** cross-runtime vectors in both Python and C# together;
5. `MarketSpreadInput` + Tool Gateway handler + `OddsMarketClient` verification;
6. the 22-category fixture matrix, controller-host rejection matrix, and full no-drift
   proofs.

Steps 3 and 4 are the same class of work as the earlier producer-replay correction, which
consumed a full dedicated slice on its own. Attempting all six in one pass would risk
landing a half-versioned state, which is precisely the failure mode this system has already
paid for twice.

## proposed decomposition (proposal only; nothing authorized)

- **Pass 1 -- binding authority (offline).** One versioned pure authority + policy profile
  implementing the eight qualification rules and the closed
  `exact_start | bounded_start_tolerance` method, with fixture categories 1-9. Reused by
  the events gate and paid adapter. Bump only the screen bundle and events artifact;
  regenerate their vectors. No propagation yet, so nothing downstream can misread it.
- **Pass 2 -- propagation (offline).** Carry the verified binding through envelope ->
  planner -> flight plan -> provenance with a coordinated version bump and full vector
  regeneration, plus fixture categories 10-13 and 21-22, and the market-backed vs
  planned-market-missing distinction.
- **Pass 3 -- execution enforcement (offline).** `MarketSpreadInput` binding member, Tool
  Gateway handler, `OddsMarketClient` independent verification and fail-closed reasons,
  controller trust boundary, fixture categories 14-20, and the legacy no-binding
  compatibility proofs.

Only after Pass 3 should any paid WI-0036 market-backed flight be considered, and that
would still need its own separate authorization.

## posture

Zero StatsAPI, Odds `/events`, Odds `/odds`, model, Tool Gateway, AgentRun, capture,
settlement, reconciliation, scheduling, migration, or database activity. No live service
started. No integration, push, or remote mutation. **$0.**

The persisted gamePk 823438 canary is untouched: not backfilled, not rewritten, not
retroactively rejected, and not promoted to settled evidence. It remains readable historical
pre-binding evidence.

## verification

`dai` full DevCore.Api.Tests **1540/1540** (1536 baseline + 4 new characterization tests);
`git diff --check` clean; the only changed file is the new test file; no production source,
contract version, prompt, decision, confidence, lean, posture, buyer, or matching code was
touched.
