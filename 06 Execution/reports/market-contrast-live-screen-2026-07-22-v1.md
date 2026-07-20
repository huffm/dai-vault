---
title: "Market-Contrast Live Screen 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0035 Slice 3: one governed live market-contrast screen + planner pass-2 replay"
repos:
  dai: "no code change (live execution of integrated slice-2 surface)"
  dai-vault: "docs-only (this report; branch wi/0035-market-contrast-live-screen-2026-07-22)"
tags:
  - system-development
  - evidence-operations
  - sports-v1
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
  - "06 Execution/reports/market-contrast-source-adapter-slice-2-closeout-2026-07-20-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-v2-day2-2026-07-11-v1.md"
---

# market-contrast live screen 2026-07-22 v1

## authorization and withheld authority

One governed WI-0035 Slice-3 live screen for MLB target date 2026-07-22: at most one Odds
API `/odds` request (tenant 1, baseball_mlb, region us, markets h2h,spreads, <=2 credits,
zero retries, 5-minute freshness). Explicitly WITHHELD and not exercised: model calls,
generation, AgentRun creation, capture, execution, settlement, reconciliation, tuning,
market-baseline capture, scheduling, Tool Gateway / Action Network, application-data or
database writes, dependency downloads, code or test-fixture changes, push/merge/history
rewrite, and any second paid attempt. The screen and planner outputs authorize nothing;
every authority boolean is false.

## integrated ancestry (verification-only fetch)

- dai main == origin/main == `8e044a429af6c63d0e3d166c534e657757038814` (contains the
  required commit).
- vault main == origin/main == `ad57f26a8eb6aa1724dff3bb890ef9a2e8c595d0`.
- Integrated versions verified before any source call: adapter
  `market-contrast-source/1.1`, bundle `market-contrast-screen-bundle/1.1`, classifier
  `market-contrast-screen/1.2`, planner cli `daily-evidence-planner-cli/2.3`, request
  `daily-evidence-planner-request/2.1`, board `daily-evidence-board/2.2`, envelope
  `input-evidence-envelope/1.1`.

## evidence-need grounding (canonical)

pooled_calibration `discrimination_hybrid_v1`: `conclusionsAllowed=false`, failingReasons
[`discrimination_inverted`, `insufficient_market_disagreement`]. Provenance
[[gate4-evidence-readout-v2-day2-2026-07-11-v1]], sha-256
`378bf6376445c35b3882211942a48fa854ee15cd359eb3d396cd4207d7c99b3b` (matched; verdict
current -- no settled runs since the 2026-07-11 baseline). Grounds
`evidence.market_disagreement`.

## tenant scope

TenantKey 1 (configured local development tenant), verified without querying or printing
tenant data; not substituted.

## frozen pass-1 (supporting evidence, not the screen input)

- request sha-256 `0fccafb19d60d193bf4a10c9b461a70cbef9f2ff16bfc4b5a8f1e13565886899`
  (schema 2.1; chosen objective `evidence.market_disagreement`; target 2026-07-22; honest
  `input.market_contrast_screen = NO_RECORDS`).
- board canonical content sha-256
  `3da03021d3ee6c21f4a1d18bc59735f5734e52a889073264e5d2fd919704156b`; retained file bytes
  (terminal CRLF) sha-256
  `6548cf096cd7332b79158cf37aa2c4deab66627fa7c427c379cb375da668116f`.

## candidates: authorized vs actual

Authorized set (15): 822784 822873 823110 823438 823518 823761 824004 824083 824166
824327 824408 824650 824732 824896 825055. Fresh preparation observation
2026-07-20T06:57:35Z (one schedule/hydrate request): all 15 still Scheduled, on the
2026-07-22 eastern date, >=60-minute margin, unique identities -> slate accepted all 15
(none added; none outside the set). Source sha-256
`a45832cdc8e4a10316fc4444c1b3147eecb79f4f839f1796510ed2956cb41f4a`; slate sha-256
`37ee908b353ec241d5ab0ae4c5605d1c86278881ffdae092c5df296ae6e8651f`.

## preflight (free) -- exit 0

status `completed_preflight_no_paid_call`; bundle sha-256
`c6850cc01c17c54fff837d83d0598a0ed1ea1e125dc3944e3f467616537454f3`. Ledger: odds
attempted false / 0 requests; db reads 1 / writes 0; schedule attempts 1; pitcher attempts
within bound; zero timeouts; active-run evaluated. 11 candidates screenable with margin; 4
free-preblocked `starters_not_announced` (822784, 822873, 824327, 824896 -- home starter
TBD). All admission checks passed; secret absent from captured output.

## paid attempt (one Odds request) -- exit 0, terminal status `completed`

bundle sha-256 (file bytes) `1e85c71f7cbb1296499fc9e1fb936c11194bdf8c1f81facbef3eb728f8923ed9`;
attempt id `19c4b6c62249`. Call ledger: statsapi schedule 1 / pitcher 22 / memo 0 /
failures 0; **odds requests 1**; db reads 1 / writes 0; timeouts none. Quota headers
`x-requests-last=0`, `x-requests-used=282`, `x-requests-remaining=218` (all finite
nonnegative numerics; exact strings preserved). **Charged credits: 0** (`x-requests-last=0`,
within the 2-credit ceiling; a zero-cost response remains terminal and is not rerun).

## screen classification summary

No candidate reached an includable classification -- a truthful terminal screen with no
eligible cohort. Every candidate is `blocked` (tier none): 4 `market_not_evaluated`
(the free-preblocked starter-TBD games, market skipped) and 11 `source_unavailable` --
the Odds `/odds` response for the 2026-07-22 eastern bracket contained no event matching
the authoritative provider teams AND scheduled-start instant (`no_market_match`), which
under classifier 1.2 projects the real source condition (unavailable), never a derived
readiness reason. Zero credits were charged, consistent with the provider returning no
priced events in the requested future bracket. Actual DAI/market disagreement remains
`unknown_until_generation`; decision-time market baseline `unknown_until_generation_capture`.

## deterministic pass-2 replay (schema-1.1 terminal bundle)

replay run twice to separate no-overwrite destinations, both exit 0, byte-identical:
pass-2 request sha-256 `32bb0851a5bc69a0406ad3b9dcb0923a84104ced0d3e032811c385d1f27ffefc`
(reported pass-1 hash and bundle hash equal the frozen inputs; 15 envelopes carried from
the bundle, none hand-composed; cross-date/cross-identity protections active). plan run
twice, both exit 0, byte-identical: pass-2 board sha-256
`dfb09b2a2dc5bea4295939d2495c38505b089dd5e447d244f73814595e4fecf4`; both parse and both
pass the CLI board-envelope validator (`interior_validated:false`).

## pass-2 outcome and ordered pools

`EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE`; primary pool [] and reserve pool [] (no
grounded includable market-contrast envelope, so `input.market_contrast_screen` aggregates
to `NOT_EVALUATED`); eligible candidate count null; all safety-ledger values false. A
valid honest no-cohort outcome -- rank rescued nothing and no candidate lacking grounded
current evidence entered a pool.

## verification

Pre-source: build (no restore) 0 errors; targeted C# (MarketContrastSourceAdapter/
MarketContrastScreen/MarketDepth/PromptTraceService) 78/78; full DevCore.Api.Tests
1363/1363; planner + cli 87/87; full agent-service 562/562; diff --check clean; the only
build warnings are the pre-existing NU1903 package advisory (no csproj change). Post-run:
targeted C# market-contrast 49/49; planner + cli 87/87; all canonical JSON parsed by two
standard parsers; deterministic replay and board hashes equal. Artifact scans clean:
no secret, no machine path in json, no NaN/Infinity, no stale schema, no
`authorized:true`, no tenant field in the slate. `secret_present_in_output: false`. Raw
artifact directory is outside both git repositories.

## all-false authority and side-effect ledger

Every authority boolean false in preflight bundle, screen bundle, and both pass-2 boards.
Application-data writes: 0. Database writes: 0. Model calls / generation / AgentRuns /
capture / settlement: 0. No staging leftovers; no recovery artifact (publication
succeeded). No push / merge.

## protected / classified path integrity

Byte-identical and unstaged open -> close: dai `DevCore.Data.csproj` 63ef2488 (1099b);
vault `.obsidian/graph.json` b3d68588, `CLAUDE.md` 9127e464, preflight manifest 68948ebd,
system-state synopsis 25835e6c; `Welcome.md` remains deleted.

## workspace artifacts (uncommitted, outside both repos)

`<DAI_WORKSPACE_ROOT>/screen-2026-07-22/`: `statsapi-slate-source.json`, `slate.json`,
`pass1-request.json`, `pass1-board.json`, `preflight-bundle.json`, `screen-bundle.json`,
`pass2-request.json`, `pass2-board.json`, `verification/pass2-request-repeat.json`,
`verification/pass2-board-repeat.json`, `artifact-hash-and-call-ledger.txt`, and bounded
stdout/stderr captures. Raw source payloads, bundles, requests, and boards are not
committed.

## no generation / capture / execution

Nothing in this run authorized model generation, capture, execution, or settlement. The
screen produced no eligible cohort; even had it, a valid screen bundle and pass-2 board
authorize none of those.

## next operator decision

Because the paid Odds observation for a ~2-day-forward date returned no priced events (no
market match; 0 credits) and the pass-2 board is truthfully non-addressable, the natural
next governed step is a separately authorized re-run of this same one-request screen
closer to the 2026-07-22 pregame window (when h2h/spreads are posted), reusing the frozen
pass-1 and the same bounded contract. That is a new authorization, not implied here. Slice
4 (operating integration) remains deferred until a stable CLI has completed at least one
reviewed manual use with a grounded cohort.
