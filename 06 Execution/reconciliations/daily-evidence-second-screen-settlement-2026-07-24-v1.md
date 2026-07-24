---
title: "Daily Evidence Second Screen Settlement 2026-07-24 v1"
type: "reconciliation"
date: "2026-07-24"
status: "complete -- one eligible run settled incorrect via canonical identity reconcile; zero cost; no policy change"
project: "DAI"
slice: "daily evidence operation 2026-07-23 second screen (settlement phase; no work item)"
repos:
  dai: "unchanged (read-only; integrated CLIs and API invoked from main 85af96d)"
  dai-vault: "docs only; local branch ops/2026-07-23-late-slate-reevaluation, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - reconciliation
  - settlement
related:
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "06 Execution/reports/daily-evidence-second-screen-capture-2026-07-23-v1.md"
  - "06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md"
---

# daily evidence second screen settlement 2026-07-24 v1

## purpose

Settle the single frozen July 23 second-screen capture -- run
`d329433e-f36b-1410-8196-00373db4b724` on gamePk `822785` (Tampa Bay Rays @ Toronto
Blue Jays) -- against the authoritative StatsAPI final, under an explicit operator
authorization dated 2026-07-24. The July 23 capture report recorded this run as
unsettled by design; that statement is preserved as historical truth and this record
closes it.

## authorization history (two-pass)

A first 2026-07-24 authorization specified strict preflight switches
(`-RequireRegistry -RequireBackedDepth -FailOnWarnings`). That attempt stopped at
preflight with 5 blockers and no write: those switches assert registry-route fields
(`promptSource=registry`, `selectedDataRegime`, `attributionStatus=complete`, no legacy
fallback) that this live-authoritative capture structurally does not populate
(routing reason: "canary disabled; live prompt is authoritative"). The operator issued a
replacement authorization with a live-path-compatible flag set and a strict five-warning
semantic allowlist. Precedent: the July 22 canary settlement (same live path) passed
preflight with blockers 0 / warnings 5 under the same relaxed switches.

## eligibility and finality gates

- identity: run `d329433e-f36b-1410-8196-00373db4b724`, tenant 1, provider
  `mlb_statsapi`, external game id `822785`, frozen flight
  `flight-2026-07-23-paid-cohort-2`, freeze fingerprint `45796b4e...` (64 hex chars),
  lane `core`, stratum `core`, realized via `scheduled`.
- finality: `check-settlement-finals.ps1 -Competition mlb -GamePks 822785` returned
  READY exit 0 -- exactly one Final game, final (1-3). Independent StatsAPI live-feed
  re-read immediately before settlement confirmed `abstractGameState=Final`,
  `codedGameState=F`, away = Tampa Bay Rays 1, home = Toronto Blue Jays 3.
- not previously settled: preflight precheck `SingleMatch`, no outcome, no evaluation,
  exactly one active unreconciled row for the gamePk.

## live-path-compatible preflight

`preflight-settlement.ps1 -Competition mlb -GamePks 822785 -ExpectedRunPrefixes
d329433e -RequireUnreconciled` (intentionally without `-RequireRegistry`,
`-RequireBackedDepth`, `-FailOnWarnings`): exit 0, target 1 / found 1 / ready 1,
blockers 0, warnings exactly 5, all inside the authorized semantic allowlist and
nothing outside it:

1. promptSource `live` != registry
2. legacyFallbackUsed is true
3. fallbackReason present: `disabled`
4. attributionStatus `partial` != complete
5. selectedDataRegime `''` != `starter_enriched_market_backed_depth`

Independent persisted-evidence checks (all passed pre-write): routing reason
`canary disabled; live prompt is authoritative`; observed data regime
`starter_enriched_market_backed_depth`; attribution fidelity `Pass`; market consensus
side `away`, book count 9, market agreement true; exclusion reason null; captured lean
`away` @ 0.75; flight id and nonempty freeze fingerprint present; no outcome or
evaluation.

## settlement execution (canonical path)

Order per settlement-readiness-discipline-v1: finals guard, independent StatsAPI
re-read, read-only preflight, then exactly one identity reconcile POST. No direct
database write, no alternate writer.

- write path: `POST /api/agent-runs/reconcile` (single canonical writer;
  `writer_path=identity_reconcile`).
- request residue: sourceProvider `mlb_statsapi`, externalGameId `822785`,
  outcomeStatus `home_win`, homeScore 3, awayScore 1, source `statsapi_final`,
  sourceRef `gamePk 822785 final away 1 home 3`, notes carrying the settlement date,
  run id, gate summary, writer path, live-path routing reason, observed data regime,
  and attribution fidelity (complete residue per the canonical reconciliation residue
  contract v1).
- response: HTTP 200, `matchKind=SingleMatch`, matchedRunIds exactly
  `[d329433e-f36b-1410-8196-00373db4b724]`, evaluatedRunId the same run,
  `evalStatus=incorrect`.
- evaluation row `5a46433e-f36b-1410-8198-00373db4b724`: lean `away`, winning side
  `home`, evaluated 2026-07-24T08:18:01-04:00.

## post-write verification (persisted reads)

- target row: outcome `home_win`, resultSide `home`, scores home 3 / away 1,
  settlementSource `statsapi_final`, sourceRef and notes nonblank and exact,
  reconciledAtUtc stamped, exclusion reason still null.
- unchanged on the target run: lean `away`, confidence 0.75, market agreement true,
  book count 9, flight id, freeze fingerprint, evidence stratum `core`, observed data
  regime, attribution fidelity `Pass`, selection lane, realized position.
- cross-run integrity: full rows read diffed before vs after -- exactly one row
  changed (the target); no unrelated run changed.
- count deltas on `prompt-route-calibration/rows` (verified deltas preferred over
  assumed absolutes): total rows 304 -> 304 (unchanged); reconciled 159 -> 160 (+1);
  active unreconciled 145 -> 144 (-1); valid settled (outcome present, exclusion null)
  156 -> 157 (+1); correct-lean count 81 -> 81 (unchanged -- this settlement is an
  incorrect evaluation, so no correctness gain). The prior authorization's nominal
  138/60 baseline belongs to a differently-filtered calibration read; the live
  endpoint snapshot above is the audited authority for this operation.
- Gate 4 classification refreshed truthfully by the new incorrect evaluation; no
  threshold, policy, or configuration change.

## cost and runtime boundaries

- zero paid provider calls; zero credits; no new capture, prediction, source, schema,
  migration, or configuration change.
- runtime started for this operation and stopped at close: Docker Desktop engine, the
  `devcore-sql` container, and the platform API on port 5007. Agent service,
  frontends, and model clients never started.
- dai repo untouched (main `85af96d`, clean). The preserved
  `wi/0035-provider-event-binding` worktree and its six dirty paths untouched.
- runtime evidence (preflight manifests, before/after row snapshots, reconcile
  response, evaluation read) retained outside both repos under the session scratchpad
  `<scratch>/settlement-822785/`.

## outcome

The day's strongest-disagreement capture (0.0360 gap, market-agreed away) settled
incorrect: the market-agreed away lean lost 1-3. One more valid settled sample; no
correctness gain; calibration evidence value intact either way. The July 23 capture
report's "unsettled" statement stands as accurate history; this record supersedes it
for current state.
