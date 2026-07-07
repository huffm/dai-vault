---
title: "HANDOFF: Evidence Acquisition Cadence Proposal v1 -- DONE (2026-07-07)"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- proposal shipped; authorizes nothing; awaiting operator approval decision"
project: "DAI"
slice: "Evidence Acquisition Cadence Proposal v1"
related:
  - "06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-sufficiency-projection-2026-07-07-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
---

# HANDOFF: evidence acquisition cadence proposal v1 (2026-07-07)

## 1. objective

docs-only proposal converting Gate-4 Evidence-Sufficiency Projection v1 into a
disciplined, approval-gated capture rhythm: mornings/week, run + spend ceilings, slate
go/no-go, stop conditions, settlement rhythm, and re-projection checkpoints.

## 2. outcome

COMPLETE. proposal shipped:
`06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md`.
recommended cadence: 2 capture mornings/week in the 10:00-13:00 ET window, up to 6 runs
per qualified morning, weekly ceiling 12 runs; target 5 six-run cohorts (~2.5-3 weeks best
case) with forced re-projection at marketDisagreementN 7 and 10. budget: ~$0.009/week
estimated, $0.10/week hard worst-case bound (2 cohorts x existing $0.05 cap); expected
total ~$0.021 at observed yield, sensitivity ~$0.011-$0.071 -- all marked approximate
(durable per-run cost sink still missing). approval model: option A per-capture explicit
approval (default, matches hold/no-spend posture) or option B standing weekly ceiling +
per-capture go/no-go; under both, no scheduler and no automatic spend. core language
preserved verbatim. **the proposal authorizes nothing by itself.**

## 3. repo state before / after

- dai before and after: main @ `dbda7a8`, 0 ahead / 0 behind, dirty only on the phantom
  `DevCore.Data.csproj`. NO changes, NO commit, NO push.
- dai-vault before: main @ `c8fbbe4`, 0/0, untracked = known manifest + synopsis noise.
- dai-vault after: proposal + this handoff committed and pushed (sha in the session
  closeout); same untracked noise retained, not committed.

## 4. services used

none required and none touched this slice (git + file reads only; no HTTP calls, no
model calls). devcore-sql / DevCore.Api remain running from the earlier slices.

## 5. work performed

- skills gate run (dai-skill-router): selected slice-runner doctrine, dai-agent-handoff,
  verification-before-completion, grill-with-vault posture; nothing missing.
- phase 0 baselines: dai `dbda7a8` clean-except-phantom; dai-vault `c8fbbe4` 0/0.
- read-first set completed: projection report + handoff (authored this session, in
  context), gate-4 readout, cohort-selection-and-run-discipline-v1, frozen-cohort-slate
  template, preflight-settlement.ps1 (read in the settlement slice), operating context
  pack (section 1 treated as volatile posture; sections 2-8 doctrine -- confirmed spend
  approval-gating, no-scheduler-by-design, WNBA-for-volume contraindicated, no-build
  list).
- phase 1 cadence inputs extracted from the projection verbatim (no recomputation; report
  internally consistent).
- phases 2-5 composed into the proposal: window rationale (v1 late-run failure vs v2
  morning success; TBD-starter lower bound), 2x6x12 weekly shape, 5-cohort target with
  checkpoint interlock (n=7 ~ 2 cohorts, n=10 ~ 5 cohorts at observed yield), budget
  estimate-vs-cap distinction, A/B approval model, GO/NO-GO checklist, 9 stop/pause
  conditions incl. gate-TRUE merit verification tied to the projection's bottom-decay
  fragility, settlement/readout loop.
- phases 7-9: validation, vault-only commit, push.

## 6. files changed

dai: none.
dai-vault (one commit):
- `06 Execution/reports/evidence-acquisition-cadence-proposal-2026-07-07-v1.md` (new)
- `06 Execution/reports/evidence-acquisition-cadence-proposal-handoff-2026-07-07-v1.md`
  (this file)

## 7. db writes / side effects

0 db writes, 0 reads against the api this slice. no service state changed.

## 8. paid calls / cost

0 paid model calls, $0.00.

## 9. validation proof

- authorization check: proposal states "authorizes nothing" in frontmatter, section 1,
  and section 9; core weekly-ceiling language preserved verbatim twice (sections 1, 5).
- no scheduler/background/auto-settle anywhere; option B explicitly re-affirms
  human-initiated capture.
- no tuning/gate-edit recommendations; criterion fragilities referenced only as inputs to
  a separate interpretation note.
- every cost figure tied to the documented 07-06 anchor and marked approximate; the
  $0.10/week bound derives from the existing $0.05/cohort guardrail, not new authority.
- "candidate edge signal" language used; explicit no-edge-claim line in section 9; no
  buyer-facing performance claims.
- dai git status: phantom csproj only (no code changed). no db writes (no api calls made).
  no paid calls (agent-service untouched).

## 10. what did not change

dai repo, database, gates, criterion, thresholds, prompts, routing, confidence, models,
buyer copy, registry flags (default-off), posture (hold/no-spend stands until the
operator approves a capture under option A or B).

## 11. open issues

- the cadence needs an operator decision: approve option A (per-capture) or option B
  (standing weekly ceiling), or decline. nothing runs until then.
- durable per-run cost sink still missing -- every cost number in the proposal carries the
  approximate caveat; Durable Cost Evidence v1 removes it and fits between cohorts.
- criterion fragilities (bottom-decay TRUE path; third-region adjacency) remain flagged
  for a separate Criterion Interpretation Note; they do not block cadence but must be
  checked whenever gate 4 moves.
- operating context pack section 1 posture paragraph is stale (predates settlement); its
  own volatility rule covers this -- re-verify live, do not edit casually.

## 12. recommended next slice

- **if the operator approves the cadence: the next approved backed_depth capture cohort**
  (frozen slate discipline, section 6 GO/NO-GO, 10:00-13:00 ET window), then its
  settlement + readout slice.
- **if not approved: Durable Cost Evidence v1** (no cohort is in flight -- the 07-06
  cohort is fully settled -- so the code-change window is open now).
- if criterion fragility becomes the bottleneck first: Criterion Interpretation Note v1.

## 13. suggested next prompt

if approved (option A example): "SLICE: Backed-Depth Divergence Capture v3 (PAID,
approval-gated). approval: PAID_CAPTURE_APPROVED=true, GAME_DATE=<date>, MAX_PAID_RUNS=6,
COST_CAP=$0.05, divergence prefilter gap <= 0.10 (secondary 0.10-0.15 labeled). run in the
10:00-13:00 ET window per evidence-acquisition-cadence-proposal-2026-07-07-v1.md section 6
GO/NO-GO. freeze the slate (frozen-cohort-slate-template-v1) before any paid call; canary
process-scoped only; capture only, settlement is the next-morning slice; end with capture
report + 13-section handoff."

if not approved: "SLICE: Durable Cost Evidence v1. mode: code-allowed (dai), TDD,
no paid calls, no captures in flight. objective: persist per-run model cost evidence
durably (JSONL sink or db column per the 07-06 interim audit ranking #2) so cost is no
longer stdout-only estimate; agent-service tests green; no decision-behavior change;
13-section handoff."
