---
title: "V2 Accelerated Capture -- day 2 (2026-07-10), 8/8 registry-routed, first deliberate divergence"
type: "evidence-report"
date: "2026-07-10"
status: "complete"
project: "DAI"
slice: "V2 Day-1 Settlement and Day-2 Capture"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - capture
  - calibration
  - cohort
related:
  - "06 Execution/plans/v2-accelerated-capture-cadence-2026-07-09-v1.md"
  - "06 Execution/plans/v2-day1-settlement-day2-capture-slice-2026-07-10-v1.md"
  - "06 Execution/reports/v2-accelerated-capture-day1-2026-07-09-v1.md"
  - "06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
  - "02 Platform/system-development/work-items/WI-0005-starter-retrieval-caches-transport-failures.md"
---

# v2 accelerated capture -- day 2

Second and final capture day of the authorized 2-slate-day cadence. Executed in-window,
settlement-paired (the day-1 cohort was settled first, per the runbook's ordering rule).

## 1. authorization and window

- window: **10:00-13:00 ET**. slate frozen 14:25:56Z (10:25 ET); first generation ~10:47 ET.
  Earliest first pitch on the slate: 18:40 ET. Every selected game strictly pre-game.
- cap: 8 eligible runs. spend cap $0.05/day. model gpt-4o-mini.
- no-backfill directive: binding. 15 of 15 games were eligible, so the cap bound, not the floor.
- registry canary `DAI_MLB_REGISTRY_PROMPT_CANARY=1` set **process-scoped** on the
  agent-service child only, explicitly approved by the operator for this capture. `.env` never
  written (verified 0 occurrences before and after); parent shell env never carried the flag;
  the flag died with the process at shutdown.

## 2. screen -- and a screen defect that would have corrupted the cohort

First screen (15 candidates, rapid succession) reported 5 `identity_unmatched` + 1 error, and
would have ranked the cohort on the surviving 9.

That was **wrong**. `MlbStarterClient` fails soft on a transport error and the caller caches the
null result for 30 minutes, so a transient network failure is stored as, and is
indistinguishable from, "no probable starter announced". Retrying with backoff only re-read the
poisoned cache.

After restarting `DevCore.Api` (clearing the in-process `IMemoryCache`) and re-screening
**serially with pacing**, all six recovered: `matched / enriched / backed_depth / 9 books /
eligible`. **6 of 15 candidates (40%) had been false-negative**, including 824493 CHC@CIN —
the narrowest de-vigged gap on the board, i.e. rank 1.

The paced 15-of-15 re-screen is the authoritative screen. The original incomplete screen was
discarded and never used for selection. Defect registered as
[[WI-0005-starter-retrieval-caches-transport-failures]] (BACKLOG, NOT AUTHORIZED, not fixed here).

**Rule learned:** a screen that ranks paid candidates must treat a transport failure as a hard
error, never as a data verdict.

## 3. eligibility

Criteria (operator's, all required): `identityStatus=matched`; starter `enriched`; market
`backed_depth` with a clear consensus side; `bookCount >= 5`; no pre-existing active run on the
gamePk; strictly pre-game.

- considered: **15** (full MLB slate, all `Scheduled`, all with both probables)
- eligible: **15** | excluded: **0**
- duplicate-active risk: **0 of 15** had any pre-existing active run (verified against `/rows`)
- de-vigged gaps from ONE the-odds-api h2h slate read (per-book devig `h/(h+a)` from decimal
  odds, median across books, gap `= |2*devigHome - 1|`) — same method as day 1.

## 4. selection (ranked by narrowest de-vigged gap, then bookCount) and captured runs

| rank | gamePk | matchup | run | lean | conf | consensus | devig gap | market | guard | class |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 824493 | CHC @ CIN | 549d433e | away | 0.75 | away | 0.0209 | agree | Pass | MarketAligned |
| 2 | 822955 | SEA @ TB | 599d433e | home | 0.75 | home | 0.0209 | agree | Pass | MarketAligned |
| 3 | 823278 | TOR @ SD | 5d9d433e | home | 0.75 | home | 0.0262 | agree | Pass | MarketAligned |
| 4 | 823845 | CLE @ MIA | 609d433e | away | 0.70 | home | 0.0444 | **opposed** | Pass | **DeliberateDivergence** |
| 5 | 824252 | PHI @ DET | 659d433e | home | 0.75 | home | 0.0537 | agree | Pass | MarketAligned |
| 6 | 823357 | MIL @ PIT | 6c9d433e | home | 0.75 | home | 0.0646 | agree | Pass | MarketAligned |
| 7 | 823685 | LAA @ MIN | 709d433e | home | 0.75 | home | 0.0891 | agree | Pass | MarketAligned |
| 8 | 823604 | BOS @ NYM | 729d433e | home | 0.80 | home | 0.1038 | agree | Pass | MarketAligned |

All 8: `promptSource=registry`, `selectedPromptVersion=v2`,
`selectedPromptRecipeId=mlb.pregame.analysis.starter_enriched_market_backed_depth.v2`,
`selectedDataRegime==observedDataRegime==starter_enriched_market_backed_depth`,
`attributionStatus=complete`, `legacyFallbackUsed=false`, 64-hex `assembledHash`,
`bookCount=9`, `evidenceRichness=2`, `advertisedStrength=High`, `exclusionReason=null`,
no outcome/evaluation (unsettled by design).

Dropped **by rank** (eligible, beyond the cap — not by relaxation): 822878 gap 0.1212;
824817 0.1568; 823200 0.1624; 822709 0.1859; 823033 0.1930; 824575 0.1980; 823927 0.3870.

Run-id suffix shared: `-f36b-1410-817b-00373db4b724`.

## 5. the finding: first deliberate divergence produced by a capture

**823845 CLE@MIA** — DAI leans away (Guardians, conf 0.70) against a 9-book home consensus
(Marlins). Persisted guard fields, verbatim:

```
attributionFidelityStatus = Pass
attributionFidelityReason = prose_acknowledges_market_opposition
divergenceInterpretation  = DeliberateDivergence
```

Every previously persisted market-opposed row in the corpus — all six in the taxonomy audit,
plus 823281 discovered by the guard in legacy rows — is an `AccidentalDivergence`: the artifact
misread the market as agreeing with its own lean. This row is the first where the model **names
the market-favored team and explicitly acknowledges opposing it**, which is precisely the
behavior Prompt Market Context Hardening v1 was built to produce (team-named consensus,
de-vigged both-side probabilities, required market-vs-lean acknowledgment).

Discipline, per taxonomy section 7.6: it is a **market-opposed row** and a deliberate
divergence. **It is not yet a candidate edge signal** — `CountsAsCandidateEdge` credit is a
settlement-time property, and this run is unsettled. One row establishes nothing about edge.
When it settles (07-11) it becomes the **second** deliberate ledger entry (after 823281,
currently 0/1) and moves the market-opposed ledger from n=7 to n=8 (readable at 10).

## 6. cost and provider calls

- paid model calls: **8** (gpt-4o-mini, one `create()` per run)
- estimated model cost: **$0.00568** of the $0.05 daily cap (~11%); metering estimate, not
  billing truth — the durable per-run cost sink is still missing (Durable Cost Evidence v1).
- the-odds-api: 15 screens (first, poisoned) + 6 retries + 6 re-screens + 15 paced re-screens
  + 2 h2h slate reads + 8 generations. The cache defect roughly **doubled** the screen cost of
  this cohort; that waste is attributable to WI-0005, not to the ranking method.

## 7. post-capture verification (persisted, not inferred)

- rows: 294 -> **302** (+8, exactly the selected set); 8 unique gamePks
- **duplicate-active gamePks: 0** at pre-capture, after every run, and at closeout
- all 8 active (`exclusionReason=null`) — capture-cadence runs ARE the intended prediction rows
  and stay active per the capture-closeout rule
- **zero diagnostic/retry runs were created**, so no closeout exclusion was owed; the canary was
  a selected slate game (rank 1), not a throwaway
- guard: **Pass 8 / Unclear 0 / FAIL 0**
- corpus guard after: Pass 88 / **FAIL 10 (unchanged)** / Unclear 204 — no v2-era run has
  produced an attribution mismatch across either cadence day, against the frozen v1 baseline of
  Pass 72 / FAIL 10 / Unclear 203
- confidence spread: 0.75 x6, **0.80 x1**, **0.70 x1** — the 0.80 row feeds the starved
  `gte_0.80` region; the 0.70 row feeds `0.70_0.74`
- lean: home 6 / away 2
- no settlement performed today (games first-pitch this evening); Gate 4 unchanged by capture

## 8. hard stops -- none triggered

1. duplicate-active > 0 — not triggered (0 at all sweeps)
2. guard FAIL on a v2 run — not triggered (8 Pass)
3. unexplained registry fallback — not triggered (`legacyFallbackUsed=false`, `fallbackReason=null` on all 8)
4. missed closeout diagnostic exclusion — not applicable (no diagnostic runs created)
5. spend cap — not triggered ($0.00568 of $0.05)

The canary (rank 1) was fully verified against registry/v2/recipe/regime/attribution/fallback
before runs 2-8 were generated.

## 9. what did not change

No prompt text, model, temperature, token limits, confidence behavior, decision behavior,
source-readiness semantics, source-depth rules, capture-ranking rules, market-agreement
derivation, reconciliation semantics, calibration formulas, Gate 4 thresholds, conclusion
gating, schema, migration, buyer contract, tenant/billing, Tool Gateway, or registry allowlist.
Registry routing remains **default-off**: the flag lived only in the agent-service child process
and died with it. `dai` code unchanged (`bb10c3c`, csproj phantom only).

The registry path was used only after the assembler proved byte-identity with the live prompt —
`attributionReason` on every run reads *"registry recipe selected and proven byte-identical to
live for regime starter_enriched_market_backed_depth"*. The model received the same bytes it
would have received with the flag off.

## 10. next

Settle this cohort on **2026-07-11** once all 8 are Final (finals guard -> strict preflight ->
identity `/reconcile` with full residue), then produce the final gate-4 readout and the
Hardened-Regime Baseline Measurement v1. Capture authorization **ends** with that wrap.

Settlement of 823845 is the one to watch: it converts the first captured deliberate divergence
into ledger evidence, whichever way it lands. A deliberate divergence that loses is still
evidence; the readout must not treat a correct one as edge at n=1.
