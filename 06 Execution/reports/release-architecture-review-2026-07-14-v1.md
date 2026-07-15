---
title: "Release Timeline, Architecture Runway, and Multisport Sequence Review v1 (2026-07-14)"
type: "evidence-report"
date: "2026-07-14"
status: "complete"
project: "DAI"
slice: "Release Timeline, Architecture Runway, and Multisport Sequence Review v1"
repos:
  dai: "unchanged (read-only inspection at 85a8831)"
  dai-vault: "docs-only"
tags:
  - architecture
  - release
  - planning
  - review
related:
  - "06 Execution/plans/v1-to-v2-release-sequence-v1.md"
  - "06 Execution/plans/competition-capability-matrix-v1.md"
  - "06 Execution/plans/cloud-and-multisport-runway-v1.md"
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
---

# release timeline, architecture runway, and multisport sequence review v1

Reviewed against dai `85a8831` (all three V1 critical-path WIs integrated 2026-07-14) and
vault `a737d4e`. Companion documents carry the sequence, matrix, and runway detail; this
report carries the verdicts.

> **OPERATOR DECISION 2026-07-15 (supersedes the commercial-activation statements in
> this report).** This report's review verdicts stand as the 2026-07-14 record, but the
> following statements are superseded wherever they appear below (sections 1, 7, 10):
> outreach starting 2026-07-15, prospect-contact quotas, demonstration conversations,
> sample delivery, and 2026-08-07 as a paid-pilot commitment. Current posture: outreach
> is deferred; prospect and buyer contact are not authorized; free sample delivery is
> not authorized; real buyer delivery is not authorized; no commercial action is
> inferred from technical readiness; commercial activation requires a separate explicit
> operator decision. August 7 remains the earliest planning target for a paid private
> pilot if commercial activation is separately authorized -- it is not an active
> commitment while outreach, buyer contact, and real delivery remain deferred. A
> passing RC verdict proves operational readiness; it does not authorize outreach,
> buyer contact, payment collection, cloud deployment, multisport implementation,
> automation, or additional feature work. The governing plan is
> `06 Execution/plans/v1-to-v2-release-sequence-v1.md` section 0.

## 1. executive verdict

**Timeline: KEEP WITH CORRECTIONS.** The engineering side of V1 is finished fifteen days
early and is genuinely strong. The commercial side has had ZERO flow: the buyer-validation
brief has been ACTIVE since 2026-07-06 with send-ready outreach copy and not one prospect
has been contacted. The dominant system constraint is **buyer acquisition**, and nothing in
the remaining plan requires waiting to attack it.

Corrections (all additive, no dates slip):

1. **Outreach starts immediately (2026-07-15)** -- during the All-Star break, before and
   in parallel with the RC drill. Outreach is a conversation, not a delivery; it needs no
   RC verdict, no settlement, no code.
2. **The RC drill's named candidates should be the 2026-07-17 TB@BOS doubleheader games
   (824766 / 824737)** -- one authorized 2-call drill then simultaneously proves the RC
   criteria AND retires the dormant doubleheader-capture evidence packet (distinct-gamePk
   creation, duplicate 409, per-game identity) with zero extra spend.
3. **A free sample needs no new spend to start**: the settled 823845 brief + recap pair is
   a complete, real demonstration artifact today. Fresh prospect-chosen samples (1 paid
   call each, cap 4) unlock after RC gate 1 passes.
4. **2026-08-07 stays the paid-pilot commitment** -- and becomes a *latest* date: if a
   prospect converts earlier and the RC verdict is in, deliver earlier.

## 2. modern software engineering assessment (lens 1)

**Genuinely reducing risk:** red-first TDD everywhere (1235 C# / 453 py / 134 vitest, all
green through 13 slices); small independently verifiable batches (each WI one bounded
change, ff-only integration, byte-identical tree proofs); falsifiable claims discipline
(Gate 4 refusing conclusions at n=15 is exemplary experimental honesty); reproducibility
(deterministic exports, snapshot tooling byte-determinism, strict planning snapshots at
0 warnings); reversibility (fail-closed states, no schema changes, retained branches).

**Producing ceremony:** the three-authorization cadence per slice (implement / integrate /
next) re-runs full suites on hash-identical commits and re-records the same facts in four
places (WI, handoff file, current-slice, MOC). For paid/write/schema work this is exactly
right. For reviewed docs-only and XS reversible code slices it is pure latency: today's
three integrations added ~2 hours of re-verification to commits whose hashes proved they
were the reviewed bytes. **Feedback gap:** every feedback loop that runs is internal;
the loop that decides whether the product should exist has never fired. The process
optimizes proof-of-correctness over proof-of-value. Recommendation detail in section 9.

## 3. systems-thinking assessment (lens 2)

**Goal:** revenue per unit of operator attention from decision-support artifacts.
**Stocks:** integrated capability (high, growing); calibration evidence (n=15 v2, frozen);
operational doctrine (fresh, untested by a stranger); buyer evidence (EMPTY); cash (zero).
**Flows and loops:**
- Loop 1 (build->test->integrate->evidence): REINFORCING and healthy -- each slice's
  tooling makes the next slice safer. Risk: it is self-justifying; it will happily consume
  all attention forever.
- Loop 2 (generation->delivery->buyer response->learning): NOT RUNNING. Zero flow since
  the product existed. This is the loop the whole system exists for.
- Loop 3 (capture->settle->evaluate->calibrate): deliberately paused (Gate 4 correct);
  resumes as a byproduct of pilot deliveries -- no separate investment needed.
- Loop 4 (features->complexity->operational burden): BALANCING loop currently held in
  check by the WI gate; watch the documentation stock (current-slice ~14k lines) -- the
  record of work is growing faster than the work.
- Loop 5 (manual operation->learning->justified automation): correctly wired; no
  automation has been built ahead of demonstrated pain.
- Loop 6 (payment->entitlement->delivery->retention): designed (ledger, Stripe-as-truth)
  but has never carried a single transaction.
- Loop 7 (new sport->new semantics->test burden->confidence): not started; the
  qualification ladder (runway doc) is the designed governor for it.
**Delays:** the fatal one is between "product ready" and "first buyer conversation" --
currently unbounded because nothing schedules it. **Leverage point:** start flow through
loop 2. One genuine prospect conversation this week changes more decisions than any
remaining engineering. **Local optimization warning confirmed:** the engineering
subsystem is being optimized while the commercial-validation loop starves.

## 4. dai doctrine assessment (lens 3)

Compliant: Stripe = truth (manual link, ledger records interpretation); manual-first
(concierge; automation gated on repeated pain); revenue-per-attention adopted as the
primary metric; frontends stripped of domain logic (WI-0011 moved buyer semantics
server-side); tenants as economic boundaries (tenant-scoped queries throughout, guard
included); no anticipated-scale abstractions built (no dashboards, no multisport
framework, no cloud).

Violations and pressures:
- **Folder-level factory/assembly-line mixing (pressure, not a violation to fix now):**
  sports niche logic (MlbStarterClient, BuyerDecisionBrief copy, market phrasing) lives
  inside the DevCore.Api platform project beside generic run/outcome/tenancy machinery.
  Acceptable for one niche; the moment sport #2 lands, the competition capability
  contract (runway doc, WI-0016 proposal) becomes the seam that keeps the factory
  generic. Do not extract earlier.
- **Agents = workers:** only one worker role exists (the analyzer). Fine -- roles stay
  generic; do not invent roles ahead of need.
- **Emerging pressure:** the buyer-ready competition list is hardcoded in the frontend
  (buyer-ready filter) rather than declared by the platform -- a capability descriptor
  (WI-0016) is the doctrinal home.

## 5. principal architecture assessment (lens 4)

**Acceptable V1 shortcuts (keep):** process-local duplicate gate on one operator host;
log-only cost telemetry; local secrets in gitignored files; synchronous generation inside
the POST request; manual provisioning; no CI (suites run locally with discipline);
`/artifact/buyer` retained though unused by the app.

**Release risks (watch during pilot):** ActionNetwork dependency uses an unofficial
endpoint with a spoofed user agent -- a provider change breaks a signal silently
(fail-soft exists; runbook R4 covers); odds key + sa password sit in local working-tree
files -- rotate before any access broadening; the AgentRuns controller is ~1,400 lines
(cohesion pressure -- split when it next changes materially, not before).

**Cloud-migration blockers (stage-2 work, none block payment):** no CI/CD; no EF
migration story verified (DB bootstrap is dev-created; managed SQL needs an explicit
migration path); prod frontend env points at http://localhost:5007; CORS whitelist is
dev-only; Entra prod registration values unfilled (fail-loud guard exists); cost log has
no durable sink.

**Multi-instance blockers (stage-4 only):** the process-local duplicate gate (needs a
db-coordinated mechanism -- filtered unique index or applock; a migration, separately
authorized); per-instance IMemoryCache for starters (consistency acceptable, but the
WI-0005 poisoning recovery assumes one instance); stdout cost telemetry.

**Multisport blockers:** no competition capability descriptor; identity/schedule/
evidence providers are MLB classes, not adapters behind contracts; prompt registry
recipes are mlb.* only; outcome vocabulary must be checked per sport (ties/OT).
None of this needs building until the second-sport gate opens.

**Unnecessary anticipatory design found: none.** The restraint to date is the
architecture's best property.

## 6. current dominant constraint

**Buyer acquisition.** Evidence: outreach copy approved 07-06, zero contacts; every
authorized hour since has gone to (excellent) internal proof. After first payment ->
**operator attention / repeat-delivery quality** (the ledger measures it). After three
weeks of repeat use -> **operator attention vs. buyer count**, which is the cloud gate.
After cloud -> **second-sport evidence availability** (NBA season timing). Only the
capability contract (long-lead, rewrite-avoiding) justifies any early motion against a
future constraint, and only at design depth.

## 7. timeline verdict detail (required analysis 1)

RC drill IS the correct next operational step -- but not the next *day's* step: the break
(07-15/16, no slates) makes outreach the only value-producing work available, and it
needs nothing from the drill. August 7 is realistic with slack. Sequential truth: sample
generation depends on RC gate 1; paid delivery depends on a paying buyer + RC verdict;
settlement gate 2 blocks only the RECAP demonstration and the RC record -- NOT outreach,
NOT samples, NOT conversion. Excessive gates: none removed for paid/write paths; the
docs-ceremony finding is section 9. Missing gate: none -- the commercial gates now exist
(sequence doc). Attention balance: to date ~95% internal proof / 5% market proof;
target through 08-07: >= 50% operator attention on the commercial loop.

## 8. key risks and roadmap invalidation

1. **No willing buyer:** if >= 10 genuine prospect conversations by 2026-08-21 produce
   zero paid conversions, STOP feature work; the validation brief's stop criteria govern
   (price, promise, or segment is wrong -- simplify or pivot; do not build cloud).
2. **Paid-but-unused:** payment without >= 3 reads/week by the buyer invalidates the
   workflow-value thesis -> fix delivery fit before any expansion.
3. **Operator burden:** > 60 min/delivery day sustained across week 2 -> automation gate
   opens early (ledger evidence, not intuition).
4. **Provider fragility:** ActionNetwork/odds coverage loss during pilot -> R4 posture;
   if persistent, the evidence-sufficiency doctrine already degrades honestly.
5. **Season math:** MLB regular season runs into late September; the pilot window is
   safe; NBA gating (late October tip-off) anchors the Q4 option, not V1.
6. **Model/pricing shifts:** metering now makes cost drift visible per run.

## 9. work-item process critique (required analysis 12)

Working well: WI gate stops scope creep; red-first evidence is real; authorization
separation on paid/write/irreversible actions is non-negotiable and stays.

Too heavy, with recommended changes (process doctrine, operator decision to adopt):
1. **Bounded-change combined authority:** for XS/S slices that are reversible, docs-or-
   presentation-only, and touch no locked layer, one authorization may cover implement +
   integrate (still red-first, still reviewed, still ff-only). Saves a full
   re-verification round per slice.
2. **Single evidence record:** the WI file becomes the sole detailed record; the
   standalone per-slice handoff file is dropped (the WI IS the handoff); current-slice
   gets a 5-line synopsis entry only; MOC keeps its one-line registry entry. Cuts the
   same fact from 4 records to 2.
3. **Explicit commercial work items:** outreach waves, samples, and pilot operations get
   tracked WIs (no-code) so buyer work competes for attention inside the same queue that
   engineering work does -- this is the concrete fix for the starved loop 2.
4. **Cycle-time on the ledger:** record authorize->integrated elapsed time per WI; if the
   median exceeds ~2 operator sessions for S-sized work, the process is the bottleneck.
5. **WIP limit 1 implementation slice at a time** (already de facto true; make it rule).
Unchanged: paid calls, external writes, reconciliation, tenant data, schema, and
anything irreversible keep the full gate structure exactly as-is.

## 10. required conclusions (1-18, compact)

1. **Aug 7 correct?** Yes -- as the latest date; earlier delivery allowed once RC verdict
   + a paying buyer exist.
2. **Remaining before Aug 7:** RC drill gates 1-2 (07-17/18, TB@BOS DH), payment-link
   test-mode dry-run, outreach (>= 10 genuine contacts), >= 1 free sample delivered, one
   conversion, live Stripe link + ledger entitlement.
3. **Outreach before RC settlement completes?** YES -- it starts 07-15, before the drill
   itself.
4. **Cloud before payment?** No. Nothing about payment requires hosting.
5. **Cloud trigger:** 08-21 gate: >= 2 paying buyers retained, OR sustained > 45 operator
   min/day, OR a converting buyer who requires hosted access. Any one suffices.
6. **First cloud stage:** stage 2 -- single-instance cloud environment; recommended
   target Azure Container Apps + Azure SQL (runway doc; provisional, one alternative
   compared).
7. **Multi-instance blockers:** process-local duplicate gate (db-coordinated uniqueness =
   migration, separately authorized), per-instance starter cache, stdout cost telemetry.
8. **Sport-generic share:** platform core (runs/outcomes/evaluation/reconciliation/
   tenancy/auth/metering/brief-recap frames) is genuinely generic; the evidence pipeline
   (identity, schedule, starters, readiness, prompt recipes) is MLB-specific behind thin
   seams -- detail + evidence in the capability matrix doc.
9. **Second sport:** NBA -- daily slate cadence mirrors the MLB operating model (fast
   qualification ladder + same concierge rhythm), buyer-ready flag and legacy NBA runs
   already exist, season tips late October exactly when the Q4 option opens. NFL third
   (weekly cadence = slow learning loop; heavier injury/depth-chart evidence).
10. **Collegiate credibility:** only after a second PRO sport passes the ladder, plus
    identity normalization at ~360-team scale, provider-coverage audit, thin-data-
    dominant posture, and tournament/conference semantics -- treated as its own release,
    never "another competition code."
11. **Next <= 6 formal WIs:** proposed (NOT minted) in the sequence doc: WI-0014 cloud
    stage-2, WI-0015 db-coordinated duplicate enforcement, WI-0016 competition capability
    contract, WI-0017 NBA qualification execution, WI-0018 buyer copy polish, WI-0019
    evidence-justified ops automation. All gated; none start in the next 30 days except
    as their gates open.
12. **Explicitly NOT in the next 30 days:** cloud build-out, capability-contract
    implementation, any NBA/NFL/collegiate code, multi-instance enforcement, automation,
    self-service/tenant onboarding, history page, registry default-ON, /metrics
    denominator change, EF tenant filter, controller refactors.
13. **Process too heavy where:** integration re-verification of hash-identical commits
    for low-risk slices; quadruplicate record-keeping; missing commercial WIs (sec. 9).
14. **Dominant constraint:** buyer acquisition (sec. 6).
15. **Committed critical path:** sequence doc, section "committed horizon" -- outreach
    07-15 || RC drill 07-17-18 -> samples -> conversion -> paid delivery <= 08-07 ->
    >= 3 deliveries/week -> 08-21 gate.
16. **Forecast (conditional on 08-21):** pilot hardening (late Aug) -> cloud stage 2
    (Sept, gated) -> capability contract (late Sept) -> NBA qualification stages 1-5
    (Oct) -- sequence doc.
17. **Q4 options:** NBA pilot; collegiate feasibility spike; tenant onboarding;
    ledger-justified automation -- options only, each behind its gate.
18. **Invalidation evidence:** section 8 items 1-4; plus any RC-drill hard failure that
    survives one re-drill, or a provider access loss with no substitute.
