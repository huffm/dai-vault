---
title: "Market Contrast Final Free Identity Gate 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "NO-GO -- exact current screenable join count 1, below the closed threshold of 4; no paid screen proposed"
project: "DAI"
slice: "WI-0034 / WI-0035 / WI-0036 composition (final free identity gate)"
repos:
  dai: "unchanged (read-only; integrated operators invoked from main 48a2931)"
  dai-vault: "docs only; local branch ops/2026-07-22-planner-pass1-free-preflight, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - observation
related:
  - "06 Execution/patterns/daily-evidence-acquisition-operating-workflow-v1.md"
  - "06 Execution/reports/daily-evidence-planner-pass1-free-preflight-2026-07-22-v1.md"
  - "06 Execution/reports/market-contrast-events-gate-observation-2026-07-22-v1.md"
  - "06 Execution/reports/market-contrast-start-instant-normalization-analysis-2026-07-22-v1.md"
---

# market contrast final free identity gate 2026-07-22 v1

## purpose and outcome

Final zero-cost provider identity/start observation for 2026-07-22, run under the unchanged
exact join predicate, to decide whether today's morning workflow justifies a separately
authorized paid market screen.

**Outcome: NO-GO.** The exact current screenable join count is **1**, below the closed
threshold of **4**. No paid screen is proposed, and today's primary market-backed
wildcard-capture path closes cleanly.

## clocks and freshness decision

| moment | UTC | ET |
|---|---|---|
| entry | `14:20:42Z` | 10:20:42 EDT |
| replacement preflight | `14:21:54Z` -> `14:21:58Z` | 10:21:58 EDT |
| `/events` observation | `14:22:10Z` | 10:22:10 EDT |

Inside the 11:15 EDT gate. The 10:05 preflight bundle
(`c58623e9…`, completed `14:05:25Z`) was **15.3 minutes old** at the decision boundary,
exceeding the five-minute limit, so it was **not reused**. Exactly one replacement preflight
ran. The replacement bundle was **13 seconds old** at the `/events` boundary.

## replacement preflight

Terminal `completed_preflight_no_paid_call`; bundle sha
`b83890d363cd81c1fde69fa921b51af5c08d8afc76934dfdb9343245aec28abe`; schema
`market-contrast-screen-bundle/1.3`; mode `preflight`; exact Pass-1 hash
`0fccafb1…`; 15 distinct identities; **13 screenable** (>= 4, so the capacity stop condition
did not trigger); Odds requests 0; StatsAPI schedule 1; database reads 1, writes 0;
authority ledger all false.

## `/events` observation

One `/v4/sports/baseball_mlb/events` request, zero retries, artifact sha
`fb29c43013095afa7e59a88ee685285b4f9127f26f0d1a82fc538aab4c2859c4`, operator
`market-contrast-events-gate-operator/1.1`, attempt id `c5f018c4dc10`.

**Zero-quota audit PASSED:** `x-requests-last` exactly `"0"`, used `282`, remaining `218` --
unchanged from both the 2026-07-20 paid screen and the 08:38 observation, independently
corroborating zero credit consumption. Provider events returned: **17**. Authority ledger 8
fields, all false. No `/odds` request; the events path has no code route to it.

## exact-match table

Execution cutoff is `scheduled_start - 60 minutes`. All candidates retained well over 60
minutes at observation time; **no execution is granted**.

| gamePk | matchup | start (UTC) | provider event | delta | screenable | ambiguity | cutoff | vs attempt 1 |
|---|---|---|---|---|---|---|---|---|
| 822784 | TB@TOR | 23:07:00Z | - | +60s | yes | unique | 22:07:00Z | same |
| 822873 | CWS@TEX | 00:05:00Z | - | +60s | yes | unique | 23:05:00Z | same |
| 823110 | CIN@SEA | 19:40:00Z | - | +60s | yes | unique | 18:40:00Z | same |
| **823438** | **LAD@PHI** | **22:40:00Z** | **111a95579587…** | **0s** | **yes** | unique | 21:40:00Z | same |
| 823518 | PIT@NYY | 17:05:00Z | - | - | no (`caller_start_mismatch`) | unique | 16:05:00Z | same |
| 823761 | NYM@MIL | 18:10:00Z | - | +60s | yes | unique | 17:10:00Z | same |
| 824004 | STL@LAA | 20:07:00Z | - | +60s | yes | unique | 19:07:00Z | same |
| 824083 | SF@KC | 18:10:00Z | - | +60s | yes | unique | 17:10:00Z | same |
| 824166 | MIA@HOU | 00:10:00Z | - | +60s | yes | unique | 23:10:00Z | same |
| 824327 | WSH@COL | 19:10:00Z | - | +60s | yes | unique | 18:10:00Z | same |
| 824408 | MIN@CLE | 22:40:00Z | - | +60s | yes | unique | 21:40:00Z | same |
| 824650 | DET@CHC | 00:10:00Z | - | -240s | yes | unique | 23:10:00Z | same |
| 824732 | BAL@BOS | 23:10:00Z | - | - | no (`starters_not_announced`) | unique | 22:10:00Z | same |
| 824896 | SD@ATL | 23:15:00Z | - | +60s | yes | unique | 22:15:00Z | same |
| 825055 | ATH@ARI | 19:40:00Z | - | +60s | yes | unique | 18:40:00Z | same |

Reversed orientation 0; unresolved 0; ambiguous 0; same-orientation team pairs 13.

## comparison with attempt 1 (08:38)

**Every cell is identical** -- same dispositions, same deltas, same single exact match, same
two preblocks, same 17 provider events, same quota counters. Nothing improved across
3h44m of elapsed game day.

Notably **824732 BAL@BOS still has not cleared `starters_not_announced`**. The earlier
analysis identified it as already start-aligned with the provider (`23:10:00Z` on both
sides); it remains blocked upstream, so it is still not screenable and cannot count toward
the threshold.

### incidental evidence for the start-instant question

This is now a **second independent observation of provider instants**, ~104 minutes after
the first, and the offsets are **stable to the second**. Stability weakens the
schedule-movement explanation for the +60s population. It does **not** revive the
rounding-equivalence hypothesis: the decisive counterexample is reproduced here unchanged --
scheduled `22:40:00Z` maps to `+0s` for 823438 and `+60s` for 824408, and scheduled
`00:10:00Z` maps to `+60s` for 824166 and `-240s` for 824650. Identical inputs still produce
different outputs, so the predicate stays as it is. No tolerance was widened.

## closed decision

Counting only candidates that are screenable in the newly admitted preflight, exactly one
provider match under the unchanged predicate, still pregame with >= 60 minutes remaining,
and unique/unambiguous:

```
exact_current_screenable_join_count = 1
paid_screen_proposal_ready = (1 >= 4) = FALSE
```

**Disposition: `DAILY_EVIDENCE_FINAL_FREE_GATE_NO_GO_INSUFFICIENT_EXACT_JOIN_CAPACITY`.**

One exact join was not reinterpreted as sufficient merely because the paid endpoint is
technically callable, and no filter was loosened to justify spending. Four exact joins would
have been only a necessary source-capacity condition anyway -- never market eligibility,
flight eligibility, or capture authority.

## ledger

Odds `/events` **1** (zero quota); Odds `/odds` **0**; StatsAPI schedule 1; database reads 1,
writes **0**; model, Tool Gateway, AgentRun, capture, generation, scheduling, settlement,
reconciliation **0**; migrations applied **0**; **$0**.

Dependency: opening state was Docker engine down and `devcore-sql` stopped; only that
dependency was started for the narrow read and the opening state was restored at close. The
web application and agent-service were never started.

## artifacts (outside both repositories)

`<DAI_WORKSPACE_ROOT>/events-gate-2026-07-22/attempt-2/`

| file | sha-256 |
|---|---|
| `preflight-bundle.json` | `b83890d363cd81c1fde69fa921b51af5c08d8afc76934dfdb9343245aec28abe` |
| `events-gate-artifact.json` | `fb29c43013095afa7e59a88ee685285b4f9127f26f0d1a82fc538aab4c2859c4` |
| `attempt-manifest.json` | `8d3b37bf37b85a4ecf7fca5265b159dfcb6bbab82faaf8398f192c20e1dd81f8` |

Plus four bounded stdout/stderr logs (both stderr empty). Attempt 1 was neither deleted nor
modified; attempt 2 was claimed empty before the first live call. No secret, API key,
query-string credential, raw provider payload, tenant payload, or machine-specific path is
committed to either repository.

## what closes and what does not

Today's **primary market-backed wildcard-capture path closes cleanly**. No paid screen,
Planner Pass 2, flight plan, freeze, realization, capture, or execution occurred or is
proposed.

An optional afternoon flight remains a **separate workflow** with its own flight id,
authorization, artifacts, and immutable freeze -- it must not mutate or extend this attempt.
The WI-0036 migration remains generated and unapplied. WI-0036 Slices 1-3 keep their
integrated status; Slices 4-6 remain deferred.

## recommended next action (proposal only)

Do not spend on this slate. The binding constraint is not budget or timing -- it is that
only one candidate joins exactly, and that number was stable across two observations nearly
two hours apart. The productive next work is offline: decide deliberately, with its own
authorization and fixtures, whether the `+60s` population represents a real identity-join
defect worth a designed fix, using the two captured observations as input. Nothing about
that decision requires a provider call.
