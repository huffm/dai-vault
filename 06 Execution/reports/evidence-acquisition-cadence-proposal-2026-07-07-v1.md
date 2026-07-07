---
title: "Evidence Acquisition Cadence Proposal v1 (2026-07-07)"
type: "report"
date: "2026-07-07"
status: "PROPOSAL -- authorizes nothing; per-cohort approval gates preserved"
project: "DAI"
slice: "Evidence Acquisition Cadence Proposal v1"
related:
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-backed-depth-divergence-2026-07-06-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
  - "06 Execution/patterns/frozen-cohort-slate-template-v1.md"
---

# evidence acquisition cadence proposal v1

## 1. purpose

convert the gate-4 evidence-sufficiency projection into a disciplined, approval-gated
capture rhythm. **this document is a proposal only.** it authorizes no spend, no capture,
no automation, no scheduler work, and no gate changes. it composes from
[[gate4-evidence-sufficiency-projection-2026-07-07-v1]] and
[[cohort-selection-and-run-discipline-v1]]; it recomputes nothing.

**a weekly ceiling is not spend authority. each capture still requires slate
qualification, preflight discipline, and explicit approval unless a separate standing
approval is granted.**

## 2. current evidence gap (from the projection, 2026-07-07)

- gate 4: conclusionsAllowed = false; failingReasons = [discrimination_inverted,
  insufficient_market_disagreement, insufficient_market_coverage].
- coverage gap: +2 covered settled directional rows (exact: 60/100 = 0.60).
- disagreement gap: +5 settled marketAgreement=false rows (5 -> 10).
- observed targeted yield: 1 settled divergence row per 6 captured runs (one hit; weak).
- observed-yield planning baseline: ~30 captured runs = ~5 six-run cohorts.
- sensitivity band: 15-100 captured runs depending on true divergence yield (5%-33%).
- estimated model cost band: ~$0.011-$0.071 total (anchor ~$0.00071/run, hand-recorded
  metering estimate -- durable per-run cost evidence does not exist yet).
- discrimination is outcome-dependent and not volume-purchasable; capture volume can grow
  the gte_0.80 sample (~0.31/run observed) but cannot force its accuracy.
- re-projection checkpoints: marketDisagreementN = 7 and = 10.

## 3. recommended cadence

- **capture window: 10:00-13:00 ET.** timing is load-bearing: capture v1 (2026-07-05)
  blocked because it ran at 17:44 ET when 13 of 15 games were already final/in-progress;
  capture v2 (2026-07-06) screened at 08:37 ET with the full slate pre-game and ~5.5h of
  start-time margin. the window must be early enough that close-favorite candidates are
  still pre-game with posted odds, and late enough that probable starters are confirmed
  (the 07-03 canary failed on a TBD starter ~21h out).
- **2 capture mornings per week** (non-consecutive preferred, so each cohort's settlement
  and readout complete before the next capture is approved).
- **up to 6 runs per qualified morning** (the validated cohort size; the 12-run per-cohort
  guardrail remains the hard cap, but 6 keeps each cohort readable and cheap to settle).
- **weekly ceiling: 12 captured runs.**
- **cohort target: 5 additional six-run cohorts** (the observed-yield planning baseline
  for +5 divergence rows). at 2 qualified mornings/week that is ~2.5-3 calendar weeks
  best case; no-go mornings stretch it. the sensitivity band (15-100 runs) is
  acknowledged, and the cadence must NOT blindly run toward 100: the n=7 checkpoint
  (~2 cohorts at observed yield) and n=10 checkpoint (~5 cohorts) force re-projection
  long before the band exhausts.
- settlement is paired: each capture morning implies a settlement slice the following
  morning (free) and a filled gate-4 readout per cohort. capture without settlement is
  not evidence (section 8).

## 4. budget ceiling (approximate; documented anchors only)

all figures are estimates from the 2026-07-06 anchor ($0.004259 / 6 runs, model_metering
public-list estimate, NOT billing truth; the durable per-run cost sink is still missing,
so every number below is approximate by construction):

- per-cohort (6 runs): ~$0.004-0.005 estimated model cost; hard guardrail $0.05/cohort.
- per-week (12 runs): ~$0.009 estimated; absolute worst-case bound = 2 cohorts x $0.05
  cap = **$0.10/week hard ceiling** (an order of magnitude above the estimate; the cap is
  the guardrail, the estimate is the expectation).
- expected total at observed yield (~30 runs): ~$0.021 estimated.
- sensitivity total (15-100 runs): ~$0.011-$0.071 estimated.
- external quota: each capture morning consumes ~8 the-odds-api units (1 slate read +
  ~7 source-readiness screens); 2 mornings/week ≈ 16 units/week against 402 remaining at
  the 07-06 capture -- months of headroom; recheck at each capture.

## 5. approval model

two safe options; **option A is the default and matches the standing hold/no-spend
posture** (operating context pack section 1/8: spend is approval-gated).

- **option A -- per-capture explicit approval.** every capture morning requires a fresh
  operator approval naming: game date, max runs, cost cap, and the divergence prefilter.
  this is the model used for every capture to date.
- **option B -- standing weekly dollar ceiling + per-capture go/no-go.** the operator may
  grant a standing weekly ceiling (recommended value if granted: $0.10/week = 2 cohorts at
  the existing $0.05 cap). even under option B:
  - a standing ceiling is NOT automatic spend;
  - every capture morning still runs the full slate qualification + preflight go/no-go of
    section 6 and stops on any no-go;
  - no scheduler, cron, or background job is authorized -- a human initiates every capture;
  - the registry canary stays process-scoped, enabled per capture, verified off after.

**a weekly ceiling is not spend authority. each capture still requires slate
qualification, preflight discipline, and explicit approval unless a separate standing
approval is granted.**

## 6. slate go/no-go criteria (per capture morning)

GO -- all must hold:

- [ ] enough pre-game MLB games remain to form a useful cohort (>= 4 close-favorite
      candidates after the divergence prefilter; below that, the morning is marginal and
      should be declined rather than filter-loosened)
- [ ] source-readiness predicts starter_enriched_market_backed_depth for each selected
      candidate (identity matched, starters confirmed non-TBD, odds posted, books >= 5)
- [ ] divergence prefilter applied: de-vigged implied-prob gap <= ~0.10 primary
      (0.10-0.15 secondary only to reach viable size, labeled); overwhelming favorites
      excluded
- [ ] expected identities stable (StatsAPI gamePk matched; no existing active run for any
      selected gamePk)
- [ ] registry canary verified default-off before start; enablement will be process-scoped
      inline env only; restoration verified after
- [ ] the slate can plausibly contribute to the failing sub-gates: candidate
      marketAgreement=false rows (close favorites) and/or covered directional rows
- [ ] run count and cost estimate within the approved ceiling (<= 6 runs, <= $0.05)
- [ ] frozen slate doc written and timestamped BEFORE any paid call
      ([[frozen-cohort-slate-template-v1]])

NO-GO -- any one blocks the morning:

- [ ] slate exhausted (games final/in-progress) or too few pre-game candidates
- [ ] source readiness cannot reach backed_depth (starters TBD, odds not posted, books
      thin)
- [ ] identity or provenance uncertainty on any selected candidate
- [ ] preflight/screening warnings that would fail under FailOnWarnings discipline
- [ ] registry flag state cannot be verified off before or restored off after
- [ ] services unhealthy (devcore-sql, DevCore.Api, agent-service startup)
- [ ] cost estimate exceeds the approved ceiling
- [ ] no plausible contribution to coverage/disagreement evidence (e.g. only overwhelming
      favorites available)

a clean no-go is a success, not a failure (capture v1 precedent). do not loosen filters
to spend the budget.

## 7. stop and pause conditions

any of these stops or pauses the cadence; resumption requires the named review:

- **two consecutive capture mornings yield zero candidate divergence rows** (at capture
  time) -> pause; re-examine the prefilter and yield assumption before the third morning.
- **preflight failure or FailOnWarnings warnings** on any cohort -> stop; failure handoff.
- **registry/canary cannot be proven restored off** -> stop everything until proven.
- **cost evidence becomes untraceable** (missing cost lines) -> pause; consider Durable
  Cost Evidence v1 before further capture.
- **row/evaluation inconsistency in /rows** (duplicate settlement, null provenance,
  unexpected exclusion) -> stop; Gate-4 Data Reconciliation / Readout Correction slice.
- **marketDisagreementN reaches 7** -> pause and RE-PROJECT (scheduled checkpoint;
  ~2 cohorts at observed yield).
- **marketDisagreementN reaches 10** -> pause and RE-PROJECT before ANY further capture
  (the numeric sub-gate is satisfied there; further divergence capture needs a fresh
  justification).
- **gate 4 turns TRUE** -> pause and verify on merits sub-gate by sub-gate before any
  celebration, tuning, or claims (readout template verdict language; note the projection's
  flagged bottom-decay fragility -- a TRUE can arrive via degradation and MUST be
  merit-checked).
- **discrimination worsens materially** (inversion delta deepening across consecutive
  readouts) -> pause for criterion interpretation or evidence review; capture volume
  cannot fix an outcome-driven inversion.

## 8. settlement and readout rhythm

captures are not evidence until settled. each cohort follows the validated loop:

1. frozen slate record before any paid call (template above).
2. capture report with guardrail proof + run table (cohort discipline pattern section 4).
3. next morning: authoritative StatsAPI finals verification (fresh, never from the
   handoff's in-progress scores -- three games flipped on 07-06).
4. pre-settlement QA: `dai/scripts/dev/sports/preflight-settlement.ps1` strict
   (-RequireRegistry -RequireBackedDepth -RequireUnreconciled -FailOnWarnings), exit 0
   required.
5. reconciliation ONLY through POST /api/agent-runs/reconcile with full provenance
   (residue contract).
6. filled gate-4 evidence readout per settled cohort (template
   [[gate4-evidence-readout-template-v1]]).
7. projection refresh at the n=7 and n=10 checkpoints (or on any stop condition), not
   after every cohort.

## 9. what this does not authorize

- no automatic spend
- no scheduler
- no background capture
- no auto-settlement
- no gate edits
- no tuning
- no model replacement
- no buyer-facing accuracy, edge, or performance claims
- no registry default-on change
- no sports scope expansion
- no WNBA-for-volume

divergence rows remain **candidate edge signal** only; nothing here licenses an edge
claim at any n.

## 10. recommended next slice

- **if this cadence is approved (option A or B): the next approved backed_depth capture
  cohort** on the next qualified morning, using frozen-slate discipline and the section 6
  checklist, followed by its settlement + readout slice.
- if approval is not granted: **Durable Cost Evidence v1** (JSONL per-run cost sink,
  ranked #2 in the 07-06 interim audit) -- it is code work, so it must run while no cohort
  is in flight, and it directly removes the "approximate cost" caveat that pervades this
  proposal.
- if the criterion fragilities (bottom-decay TRUE path; third-region adjacency) come to
  dominate planning discussions before then: **Criterion Interpretation Note v1**
  (docs-only, no criterion edits).
