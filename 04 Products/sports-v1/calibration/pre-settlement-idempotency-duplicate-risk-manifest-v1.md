# Pre-Settlement Idempotency + Duplicate-Risk Manifest v1

**date:** 2026-06-24
**status:** SYSTEM-IMPROVEMENT LANE (write-boundary hardening). Docs-only. No reconciliation, no outcome/evaluation
writes, no tuning. Read-only DB inspection only. Corpus unchanged 47/47.
**type:** prepares (does not perform) the 2026-06-25 settlement pass. Freezes the eligible run set and the write
rules. Ran through dai-slice-runner + dai-skill-router gate + dai-grill-with-vault + verification-before-completion.
**companion:** `settlement-readiness-report-and-reconciliation-boundary-v1.md` (status taxonomy + lane boundaries).

**Anchor:** This slice prepares the write boundary; it does not cross it. Tomorrow may reconcile ONLY runs frozen
here, only when Final, only per agentRunId, only once.

## 1. Idempotency guard doctrine

**Reconciliation is idempotent by `agentRunId`.** The idempotency key is the run, never the game.

Before any outcome/evaluation write for a run, the settlement pass MUST check:

- Does an `AgentRunOutcome` already exist for this `agentRunId`?
- Does an `AgentRunEvaluation` already exist for this `agentRunId`?
- Does exactly one of the two exist (partial state)?
- Is the run already classified `already_reconciled`?
- Would the proposed write create a duplicate?

Required behavior:

| pre-write state | action |
|---|---|
| both rows exist | SKIP; report `already_reconciled` (the API also returns 409 on `POST /outcome`) |
| exactly one row exists (partial) | classify `partial_reconciliation_risk` -> `unknown_requires_manual_review`; DO NOT blindly write; resolve manually |
| neither row exists AND run otherwise eligible | eligible for tomorrow's write |

Hard key rules:

- **Idempotency key is `agentRunId`.** Never use `gamePk` alone. Never use team/date alone.
- The per-run endpoint `POST /api/agent-runs/{id}/outcome` enforces one-outcome-per-run (409 if exists) -- rely on
  it, but still pre-check so partial state is caught and reported, not silently retried.
- The identity-matcher `POST /reconcile` keys on `(provider, gamePk)` and would collapse the duplicate-risk pair --
  DO NOT use it for the 823613 group (see sec 2). Use per-run `POST /{id}/outcome` only.

Baseline verified this slice (read-only sqlcmd): all 15 staged runs have outcome=0 and evaluation=0; 0 are already
reconciled; corpus 47/47. So tomorrow starts from a clean idempotency state.

## 2. Duplicate-risk handling doctrine

**Duplicate-risk does NOT mean invalid.** It means more than one active staged run maps to the same
`(provider, gamePk)`. The game identity identifies the final SCORE; the `agentRunId` identifies the evaluable
DECISION. The same score can grade two different decisions.

Rules:

- Reconcile each duplicate-risk run **per `agentRunId`** (per-run `POST /{id}/outcome`); never collapse by gamePk.
- Do NOT silently drop or merge same-game runs. If only one should count, record an explicit exclusion reason for
  the other -- do not pick silently.
- Metrics MUST state how same-game duplicate-risk runs were handled. A calibration read must NOT treat two reads of
  the same game as two independent observations of game direction -- keep them in their own cohorts and note the
  shared gamePk; the shared outcome is one event, the two decisions are separate evaluables.

**Known duplicate-risk group (frozen):**

- `d879433e` (cohort-v2, 2026-06-24, lean **home**) + `beb5433e` (200018, 2026-06-22, lean **away**)
- shared gamePk **823613**, same Cubs @ Mets game (the 06-22 postponement, rescheduled to 06-24)
- opposite leans -> exactly one is correct, one incorrect, on the same final score
- reconcile **per agentRunId** if otherwise eligible; **do not collapse by gamePk**; do not choose one without a
  documented exclusion reason
- `beb5433e` has **no captured market baseline** (its 06-22 capture did not record one): exclude it from the
  DAI-vs-market denominator while keeping it eligible for DAI-vs-actual / home / chance if otherwise valid

**Required tomorrow-handoff note:** for duplicate-risk same-game pairs, the final score may be shared, but
outcome/evaluation rows and decision metrics must be keyed per run.

## 3. Frozen staged-run manifest (15 runs; the ONLY runs tomorrow may reconcile)

Resolved read-only 2026-06-24. Identity source = `mlb_statsapi` + gamePk for all. Market = de-vigged consensus
favorite captured 2026-06-24 (per `directional-contrast-cohort-capture-v2.md`); `beb5433e` = market_missing.
outcome/eval exists = `no` for all (verified). Exclusion = none for all (the 13 prior exclusions are NOT in scope).

| # | agentRunId | gamePk | matchup (away @ home) | date | DAI lean | mkt fav | DAI-market | o/e exists | dup group | tomorrow eligibility |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | d379433e-f36b-1410-8169-00373db4b724 | 823850 | Rangers @ Marlins | 06-24 | away | away | aligned | no/no | - | eligible when Final |
| 2 | d879433e-f36b-1410-8169-00373db4b724 | 823613 | Cubs @ Mets | 06-24 | home | home | aligned | no/no | **G1** | eligible when Final; per-run only |
| 3 | da79433e-f36b-1410-8169-00373db4b724 | 824583 | Guardians @ White Sox | 06-24 | away | away | aligned | no/no | - | eligible when Final |
| 4 | db79433e-f36b-1410-8169-00373db4b724 | 824341 | Red Sox @ Rockies | 06-24 | away | away | aligned | no/no | - | eligible when Final |
| 5 | de79433e-f36b-1410-8169-00373db4b724 | 824018 | Orioles @ Angels | 06-24 | home | home | aligned | no/no | - | eligible when Final |
| 6 | e379433e-f36b-1410-8169-00373db4b724 | 824259 | Yankees @ Tigers | 06-24 | home | home | aligned | no/no | - | eligible when Final |
| 7 | e979433e-f36b-1410-8169-00373db4b724 | 822965 | Royals @ Rays | 06-24 | home | home | aligned | no/no | - | eligible when Final |
| 8 | ee79433e-f36b-1410-8169-00373db4b724 | 823367 | Mariners @ Pirates | 06-24 | home | home | aligned | no/no | - | eligible when Final |
| 9 | f579433e-f36b-1410-8169-00373db4b724 | 822719 | Phillies @ Nationals | 06-24 | away | away | aligned | no/no | - | eligible when Final |
| 10 | f779433e-f36b-1410-8169-00373db4b724 | 822798 | Astros @ Blue Jays | 06-24 | home | home | aligned | no/no | - | eligible when Final |
| 11 | fe79433e-f36b-1410-8169-00373db4b724 | 824500 | Brewers @ Reds | 06-24 | home | **away** | **DISAGREEMENT** | no/no | - | eligible when Final (high-info) |
| 12 | 037a433e-f36b-1410-8169-00373db4b724 | 823691 | Dodgers @ Twins | 06-24 | away | away | aligned | no/no | - | eligible when Final |
| 13 | 047a433e-f36b-1410-8169-00373db4b724 | 823041 | Diamondbacks @ Cardinals | 06-24 | home | home | aligned | no/no | - | eligible when Final |
| 14 | 077a433e-f36b-1410-8169-00373db4b724 | 823284 | Braves @ Padres | 06-24 | **null** | away | **NULL_ABSTENTION** | no/no | - | eligible when Final (high-info; inconclusive by contract) |
| 15 | beb5433e-f36b-1410-8167-00373db4b724 | 823613 | Cubs @ Mets (200018) | 06-22 | away | (missing) | **MARKET_MISSING** | no/no | **G1** | eligible for actual/home/chance when Final; per-run only; OUT of DAI-vs-market denominator |

### Explicit markings

1. **fe79433e** -- Brewers @ Reds -- DAI-market **disagreement** (DAI home / market away) -- **high-information**.
2. **077a433e** -- Braves @ Padres -- **null abstention** (market away) -- **high-information** (DAI declined a side).
3. **d879433e + beb5433e (group G1)** -- shared gamePk **823613** -- duplicate-risk group -- **opposite leans** --
   reconcile **per agentRunId** -- do NOT collapse by gamePk.
4. **beb5433e** -- **missing market baseline** -- market comparison ineligible unless a baseline is recovered;
   may remain eligible for actual/home/chance if otherwise valid.

### Scope rule

Tomorrow's settlement pass may reconcile **only** the 15 runs in this frozen manifest. No prior-excluded run, no
newly-captured run, and no gamePk-implied sibling may be added unless a later slice explicitly expands scope. The
manifest is the authoritative eligible set.

## 4. Verification (this slice)

- Corpus counts before == after: AgentRunOutcomes 47 / AgentRunEvaluations 47 (read-only sqlcmd; no writes issued --
  only SELECT). Zero outcomes created, zero evaluations created.
- No winners inferred; no partial scores used as performance evidence; no staged artifact altered.
- No tuning: no prompt/model/lean/confidence/source-threshold/advertised-strength/buyer-copy/market-weighting/
  artifact-semantics change. No code change (docs only); dai tree clean.
- Manifest scope: exactly 15 staged runs; the 13 prior exclusions are excluded and not listed; duplicate-risk group
  G1 documented; market-missing case (beb5433e) documented; high-information cases (fe79433e, 077a433e) marked.

## 5. Tomorrow's settlement gate + pass (DIRECTIONAL-CONTRAST SETTLEMENT PASS V1)

- **Gate: 2026-06-25T12:00:00Z or later** (cohort-v2 games settle ~2026-06-25T04:00Z+; the 12:00Z gate ensures
  StatsAPI finals are posted).
- Pass: re-check StatsAPI finality per gamePk; for each `final_ready` manifest run that passes the idempotency
  pre-check, `POST /{id}/outcome` with the StatsAPI score then `GET /{id}/evaluation`; honor G1 per-run; keep
  beb5433e out of the DAI-vs-market denominator; never guess; measure DAI vs actual/market/home/chance; keep regimes
  separate. Docs + operational data only; do not push.

## 6. What did not change

No reconciliation; no AgentRunOutcomes/Evaluations rows; corpus 47/47; no winners inferred; no partial-score
performance; no decision-engine/prompt/model/confidence/lean/source-threshold/advertised-strength/buyer-copy/
artifact change; no new slate; no code change; no push.
