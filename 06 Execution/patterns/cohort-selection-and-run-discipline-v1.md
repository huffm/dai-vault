---
title: "Cohort Selection and Run Discipline v1"
type: "pattern"
date: "2026-07-06"
status: "active"
project: "DAI"
tags:
  - calibration
  - cohort
  - capture
  - pattern
  - doctrine
related:
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-2026-07-06-v1.md"
  - "06 Execution/patterns/frozen-cohort-slate-template-v1.md"
---

# Cohort Selection and Run Discipline v1

How to select, freeze, run, and hand off paid measurement cohorts so the capture itself is
measurement-grade. Distilled from Backed-Depth Divergence Capture v1 (blocked: ran too
late), v2 (captured clean), and Cohort Integrity QA v1 (passed).

## 1. cohort selection principles

1. **Choose the measurement objective before screening.** A cohort answers one question
   (e.g. "does DAI diverge from close markets, and is it right when it does?"). The
   objective dictates the filter; never reverse-engineer an objective from available games.
2. **Freeze the candidate slate before any paid call.** The frozen slate doc (all
   considered, all excluded with reasons, all selected with market baseline) is written and
   timestamped BEFORE generation. AgentRuns count at freeze is recorded.
3. **Decision-time market data only.** The baseline that matters is what the market said
   when DAI decided. Never backfill from later odds; never re-read after generation.
4. **Identity-safe and settlement-ready or not at all.** Every selected game must have a
   stable provider identity (StatsAPI gamePk), matched via source-readiness, before spend.
5. **Avoid overwhelming favorites for divergence measurement.** A near-certain market
   carries almost no divergence information; agreement there is noise, disagreement there
   is usually error.
6. **Separate primary and secondary candidates.** Primary = fully in-filter; secondary =
   marginal, admitted only to reach a viable cohort size, labeled as such in the slate.
7. **Record every candidate, selected or excluded, with the reason.**
8. **Record every generated run**, including failures and inconvenient results.
9. **Never replace games because early outputs are inconvenient.** After freeze, the slate
   is immutable: no additions because runs agree with market, no removals because a result
   looks wrong.
10. **No tuning during capture.** Prompts, routing, confidence, and gates are frozen for
    the life of the cohort; a mid-cohort change splits the measurement.
11. **Capture and settlement are separate slices.** The capture slice writes runs; the
    settlement slice writes outcomes; neither does the other's job.

## 2. candidate scoring model

Score each pre-game candidate on these dimensions, then classify.

| dimension | measure | good |
|---|---|---|
| identity safety | source-readiness identityStatus | matched |
| source readiness | predictedObservedDataRegime | equals target regime |
| backed_depth eligibility | generationEligibleForTargetRegime | true |
| market baseline availability | odds slate has the game, h2h priced | yes |
| book count | bookmakers pricing h2h | >= 5 (9 typical) |
| implied probability gap | abs(devig home - devig away) | <= 0.10 primary; <= 0.15 secondary |
| market disagreement / mixed books | book-to-book spread on favorite | mixed books = extra divergence value |
| start-time margin | first pitch minus now | >= 1h (avoid line-freeze scrambles) |
| settlement readiness | no existing active run for the gamePk | 0 existing |
| calibration value | does this game grow the thin dimension (disagreement/coverage)? | yes |

Classification labels:

- **Primary** — passes every dimension in-filter (gap <= ~0.10 for divergence cohorts).
- **Secondary** — identity-safe + source-ready but marginal on gap (0.10-0.15); admit only
  to reach viable cohort size, and label it.
- **Exclude** — fails the objective filter (e.g. overwhelming favorite, gap > 0.15) or is
  not pre-game. Record the reason.
- **Blocker** — identity ambiguous, duplicate active run exists, market baseline absent, or
  source-thin. Never generate against a Blocker.

Do not loosen the filter to spend the budget. A clean block beats a bad cohort.

## 3. capture run discipline

Ordered; each step gates the next.

1. **Precheck repo state** — expected HEAD, known-only dirt, no unexpected drift.
2. **Verify services** — start only what the path needs; health-check and record.
3. **Verify guardrails from source** — model id, single model call per run, cost logging,
   caps (max runs, total cost), canary default-off, allowlist contents.
4. **Screen read-only** — StatsAPI schedule -> pre-game only; one odds slate read; per-game
   source-readiness. Abort if the viable candidate set is too thin.
5. **Freeze the slate** (see template) — before agent-service starts, before any model call.
6. **Enable the canary process-scoped only** — inline env on the process; never `.env`,
   never persisted config; record how it was set.
7. **Generate bounded runs** — verify run 1's provenance (promptSource=registry, regime,
   model, one cost line) before continuing; then generate the rest sequentially.
8. **Record every run immediately** — id, lean, confidence, market baseline, agreement,
   cost line, provenance.
9. **Stop on guardrail breach** — count/cost/model/route/identity deviations stop the run;
   never push through.
10. **Restore default-off** — stop the process; verify the flag is nowhere persisted.
11. **Write the capture report** — including what did NOT change.
12. **Do not settle in the capture slice.**
13. **Run the pre-settlement QA** (scripts/dev/sports/preflight-settlement.ps1) before the
    settlement slice; attach the manifest.

## 4. required cohort artifacts

Every capture produces, in `06 Execution/reports/`:

1. frozen candidate slate (pre-generation, timestamped, with baselines)
2. capture report (numbered sections incl. guardrail proof + what-did-not-change)
3. cost/model summary (per-run + total vs cap)
4. run table (id, gamePk, lean, confidence, market favorite, agreement, provenance)
5. market agreement summary (agree/disagree counts + the divergence runs named)
6. settlement-readiness manifest (preflight-settlement.ps1 output or manual QA report)
7. continuation-grade handoff (canonical 13-section)

## 5. anti-patterns

- **Spending budget because time is available.** Budget follows the measurement objective,
  not the calendar.
- **Loosening filters to create a cohort.** If the slate doesn't support the objective,
  block and reschedule (capture v1 did this correctly).
- **Adding sports for volume without settlement-safe market baselines.** Volume on the
  wrong dimension advances nothing (see WNBA finding, 2026-07-05).
- **Replacing games mid-run to chase disagreement.** Selection bias that fabricates the
  signal being measured.
- **Treating agreement/disagreement as an outcome before the final score.** Direction
  capture is not edge evidence; only settlement makes it readable.
- **Reconciling by team name.** Settlement matches on provider identity (gamePk); names
  drift, identities do not.
- **Backfilling non-decision-time market data.** Fabricates a baseline that did not exist
  at decision time; rejected permanently (2026-07-05 diagnostic).
