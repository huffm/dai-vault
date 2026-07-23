---
title: "Provider-Event Binding Slice B -- Trusted Bundle Replay and Planner Propagation 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "Slice B implemented locally and green on both runtimes; UNINTEGRATED and unavailable to live workflow; authorizes no flight or runtime use; execution retrieval remains a separate open downstream gap"
project: "DAI"
slice: "WI-0035 provider-event binding vertical, Slice B (envelope -> replay -> planner/board propagation) + WI-0036 seam-preservation correction"
repos:
  dai: "local branch wi/0035-provider-event-binding; NOT pushed"
  dai-vault: "local branch wi/0035-provider-event-binding; NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - implementation
related:
  - "06 Execution/reports/provider-event-binding-slice-a-2026-07-22-v1.md"
---

# provider-event binding slice B -- trusted bundle replay and planner propagation

## what is now true

The content-integrity-validated `provider_event_binding` (Slice A) now propagates from the
**paid market-contrast screen bundle** through `input-evidence-envelope` into the deterministic
Planner Pass-2 request and board, under an explicit frozen replay context. The guarantee Slice B
makes -- and the only one it claims -- is:

> The Pass-2 request and board were deterministically derived from the exact frozen paid bundle
> selected by the replay context, and each propagated binding equals that bundle candidate's
> strictly validated canonical binding.

No cryptographic producer identity, signature, or live re-observation is claimed. A caller
inventing both a bundle and a matching context is explicitly outside the guarantee.

## the four-part trust model (preserved distinctly)

1. **Binding content integrity** -- the strict C#/Python validators prove the wire is
   structurally and semantically valid and that its fingerprint matches its canonical content.
2. **Frozen-artifact replay** -- the bundle-hash gate proves derivation from the exact selected
   bundle. An attacker mutating both sibling copies and rehashing them still fails, because the
   actual bundle bytes no longer equal the independently frozen expected hash.
3. **Sibling consistency** -- envelope binding == candidate binding proves only that those two
   fields agree inside that one frozen artifact; it is NOT origin proof.
4. **Authority** -- the closed eight-key all-false ledgers grant nothing.

## the new contracts

| contract | version |
|---|---|
| `screen-bundle-replay-context/1.0` | **new** -- closed frozen context (expected bundle SHA, Pass-1 SHA, target date, attempt id, bundle schema, C#-origin bracket bounds, all-false ledger) |
| `input-evidence-envelope` | 1.1 -> **1.2** (carries `market_binding_evidence`) |
| `daily-evidence-planner-request` | 2.1 -> **2.2** (carries a closed `replay_reference`) |
| `daily-evidence-board` | 2.2 -> **2.3** (carries the validated binding + replay reference) |
| `daily-evidence-planner` | 2.2 -> **2.3** |
| `daily-evidence-planner-cli` | 2.5 -> **2.6** |
| current replay bundle support | **`market-contrast-screen-bundle/1.4` ONLY** |

The cascade lands in **one** `dai` commit; C# and Python constants and all cross-runtime vectors
move together. No half-versioned intermediate state exists.

## the strict replay gate (Option B is gone)

The old Option-B replay validated only version/target-date/identities/envelopes and treated
bundles 1.1-1.3 as equivalent and binding-free. Slice B replaces it with a strict frozen-artifact
gate that, before copying any envelope:

- requires an explicit `--context` (no defaults, no ambient discovery);
- computes the actual Pass-1 and bundle SHA-256 and requires exact equality with the frozen context;
- requires `market-contrast-screen-bundle/1.4`, paid mode, a `completed` terminal, the exact
  attempt id, the internal Pass-1 hash, and an all-false enclosing ledger;
- rejects preflight / not-completed artifacts as current grounding inputs;
- requires exact provider-scoped candidate-set equality with Pass 1 (no missing/extra/duplicate/
  substituted candidate);
- validates each non-null candidate binding with `parse_binding_wire` against the **frozen context
  bracket**;
- requires `qualified_binding_count == 1` iff exactly one admitted event and one validated binding;
- requires the envelope binding to equal the candidate binding canonically;
- rejects every disagreement with a stable precise error; never repairs, infers, downgrades, or borrows.

The zero-quota events-gate artifact is not reachable from this op.

## eastern bracket ownership (settled)

Python does **not** compute Eastern DST and does **not** install `tzdata`. The context bracket
bounds originate from the C# producer receipt (`OddsMarketClient.EasternDayBracket`); Python
requires exact bracket equality and independently validates UTC whole-second shape, `from < to`,
target-date agreement, and half-open membership. The 24h / 23h (spring-forward) / 25h (fall-back)
brackets are pinned as shared C#/Python vectors, so a frozen context carrying anything else is
provably not from the authority.

## grounding contract

An includable/grounded market-contrast record requires: canonical status `qualified`, admitted
count 1, qualified count 1, and one strictly validated binding (validated at grounding time against
its own bracket) -- **plus** a request-level `replay_reference`. A hand-built request cannot claim a
grounded market-contrast record without both. A bounded +/-60s binding grounds with a literal exact
count of 0. Missing, null, unqualified, invalid, or malformed binding is non-grounding with a
stable reason; no tier, rank, or novelty fact can rescue a binding failure. Null never determines
the reason by itself -- the canonical status carries it.

## the WI-0036 seam-preservation correction

Slice B's coherent bump of the **shared** daily planner to board 2.3 structurally collided with
the integrated WI-0036 flight vertical, which builds and consumes boards through the same live
`build_board` and imported board-version constant. Because the flight CLI's C#-shared
provenance vectors regenerate through that live-reproduction path, "make the flight CLI fail" would
have broken integrated C#<->Python flight parity. Under a dated operator authorization, the seam was
preserved by **decoupling**, not by breaking:

- a new **frozen `daily_evidence_planner_compat_v22.py`** -- a byte-exact snapshot of the
  integrated planner at dai HEAD `72a0347` (request 2.1 / board 2.2 / planner 2.2 / envelope 1.1),
  with a frozen request parser (`parse_request_v22`). It is historical contract support, carries no
  provider-event binding, and must never advance to 1.2/2.3;
- `wildcard_flight_plan.py` pins a WI-0036-owned `CONSUMED_BOARD_SCHEMA_VERSION = 2.2` and its 2.2
  key set (no `replay_reference`); a current board 2.3 is rejected (Slice-C-required), never
  auto-upgraded;
- the flight CLI reproduces its board through the frozen reproducer, so the integrated CLI happy
  path, realization, forged-plan rejection, publication, and the C#<->Python provenance vectors stay
  byte-identical;
- WI-0036 flight schema, binding shape, provenance, allocation, and runtime authority are unchanged.

Ownership: WI-0034 owns current planner evolution; WI-0036 owns the frozen reproducer for the
upstream version its integrated contract names; **Slice C** will deliberately migrate WI-0036 to
board 2.3 and provider-event binding.

## verification

| gate | result |
|---|---|
| full `DevCore.Api.Tests` | **1746/1746** |
| full agent-service pytest | **688/688** |
| C#/Python canonical binding + bracket vectors | byte-identical (exact / +60 / -60 ; 24h / 23h / 25h) |
| fresh-process determinism | binding + compat suites 68/68 on rerun |
| WI-0036 flight suites (incl. C#-shared provenance vectors) | **53/53**, byte-identical, board 2.2 |
| frozen compat characterization | 5 pins (versions, divergence from live, deterministic 2.2 board, current-request rejection, no binding/replay leakage) |
| strict planning snapshot | 25 work items / 6 timeline entries / **0 warnings** |
| `git diff --check` both repos | clean |
| protected hash `DevCore.Data.csproj` | `63EF2488...` unchanged |
| Option-B / stale-vocabulary scan | no Option-B production path remains (comments only) |
| binding/replay containment | confined to the two producers, the authority, the planner, the CLI, and the frozen compat + tests |
| secret / path / live-call / authority-grant scan | clean; no `*_authorized: true` on any production line |

RED evidence: the strict replay/context/gate and the binding-aware envelope/grounding tests fail
against `72a0347` (no context contract, no 1.4 gate, no `market_binding_evidence`, Option-B
replay). The version cascade first broke 104 tests (including 50 WI-0036) exactly at the shared
planner seam -- the empirical proof that the WI-0036 decoupling was necessary, not optional.

## what remains unsafe and unfinished

Execution retrieval is **untouched** and remains a separate open downstream gap (team/date fallback,
doubleheader mis-binding, no by-ID re-verification, no `MarketSpreadInput` binding member); its four
characterization tests stay green and explicitly unresolved. Slice C carries the binding to flight
provenance and migrates WI-0036 to board 2.3; Slice D enforces the binding at execution retrieval.
Planned market-missing wildcard semantics remain owned by the WI-0036 hypothesis/flight boundary.

## posture

No StatsAPI, Odds `/events`, Odds `/odds`, model, Tool Gateway, or other network call. No paid call,
AgentRun, capture, flight freeze, settlement, reconciliation, scheduling, or activation. No database
access, migration, or service start. No integration, push, PR, remote branch, or history rewrite.
**$0.** Slice B is local, unintegrated, and authorizes no flight or runtime use. The gamePk `823438`
canary remains untouched historical pre-binding evidence.
