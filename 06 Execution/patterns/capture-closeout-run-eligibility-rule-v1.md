---
title: "Capture-Closeout Run Eligibility Rule v1"
type: "execution-pattern"
date: "2026-07-08"
status: "ACTIVE -- binding on every slice that generates evidence/QA/soak/canary/diagnostic runs"
project: "DAI"
slice: "Capture-Closeout Rule v1"
tags:
  - capture
  - run-lifecycle
  - calibration
related:
  - "04 Products/sports-v1/run-eligibility-and-supersession-contract-v1.md"
  - "06 Execution/reports/run-identity-hygiene-2026-07-08-v1.md"
  - "06 Execution/reports/run-identity-hygiene-v2-2026-07-08-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
---

# capture-closeout run eligibility rule v1

## 1. the recurrence risk this rule closes

run identity hygiene v1+v2 (2026-07-08) had to retroactively exclude 22 runs across 19
gamePks. every one of them was created by a legitimate evidence slice -- real-cohort
live soak (12), starter-missing regime capture (3), canary confirmations (3), frontend
QA generations (2), first-live-capture (1), plus a rain-postponement rerun (1) -- and
every one was left ACTIVE at its slice's closeout. consequences that actually occurred:

- identity settlement blocked or endangered: MultipleMatches on any gamePk with two
  active runs (824662 needed an explicit per-run workaround; 823281 was triple-active).
- calibration denominator contamination: 823613's postponed-instance run was settled
  alongside the makeup-game run and contributed a lucky "correct" to the valid set
  (removed in v2: valid 122 -> 121, acc 0.5865 -> 0.5825).
- future SingleMatch selection risk on every affected gamePk.

automatic supersession-at-generation is explicitly deferred in the run eligibility and
supersession contract, so eligibility hygiene is an OPERATOR/SLICE obligation. this
rule makes it a required closeout step instead of a periodic cleanup slice.

## 2. which run types MUST be excluded

any run whose purpose is NOT to stand as the calibration/settlement prediction row for
its game. concretely, a run created by any of:

- capture / soak batches (request capture, shadow soak, cohort soak)
- canary or plumbing confirmations (registry canary, routing confirmation, smoke)
- frontend/UI QA generations
- load, latency, or infra diagnostics
- any rerun of a game that already has an active intended-prediction run, unless the
  rerun REPLACES it (then the OLD run is the one excluded)

reason mapping (from RunExclusionReasons):

- `diagnostic` -- evidence/QA/soak/canary runs (the common case; no provenance link)
- `superseded` + supersededByAgentRunId -- when a newer run replaces an older
  intended-prediction run (null-lean rerun, postponement remake, registry re-route)
- `invalid` -- integrity-defective runs (per the lean-encoding precedent)

a run may stay active ONLY if, at closeout, it is the single intended prediction row
for its gamePk.

## 3. when the exclusion must happen

- **at creation when known** (it almost always is: capture/QA/canary slices know their
  purpose before the first paid call): the generating slice marks the run diagnostic
  immediately after generation, in the same session.
- **at slice closeout at the latest**: no evidence-generating slice may close with its
  runs still active. "reconciliation pending" is not an excuse -- exclusion is a soft
  eligibility flag and does not touch settlement.
- postponement rule (from 823613): if StatsAPI shows a game postponed/rescheduled
  (codedGameState D / status DR) after a run was generated, the stale-instance run is
  excluded `superseded` -> the remake run when the remake is generated, BEFORE any
  settlement pass. settlement passes must never write outcomes to two runs of one
  gamePk.

## 4. required closeout evidence

every slice that generated runs against real games must include in its handoff /
current-slice append:

1. the run ids generated, each with its final eligibility state (active-intended or
   excluded+reason), and
2. the duplicate-active sweep result, which must be ZERO:

```sql
-- read-only; must return 0 rows
SELECT ExternalGameId, COUNT(*) AS ActiveRuns
FROM AgentRuns
WHERE ExclusionReason IS NULL AND ExternalGameId IS NOT NULL
GROUP BY ExternalGameId
HAVING COUNT(*) > 1;
```

a non-zero sweep is a closeout BLOCKER for the generating slice, same class as a
failing test. this protects, by construction: SingleMatch safety (one active run per
game identity) and calibration denominator integrity (valid = settled AND
ExclusionReason IS NULL never carries evidence rows).

## 5. what this rule does not do

- it does not touch runtime code, schema, prompts, models, thresholds, or buyer copy.
- it does not authorize settlement of anything (settlement discipline stays with the
  finals-readiness guard + residue contract).
- it does not replace the deferred automatic-supersession-at-generation feature; if
  recurrence happens despite this rule, that feature is the escalation path.
