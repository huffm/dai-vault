---
title: "Interim System Improvement Clinical Audit 2026-07-06 v1"
type: "report"
date: "2026-07-06"
status: "COMPLETE -- diagnosis: system healthy and correctly gated; best interim slice = Pre-Settlement QA Script v1"
project: "DAI"
slice: "Interim System Improvement Clinical Audit v1"
repos:
  dai: "unchanged (d79c38f)"
  dai-vault: "docs-only"
tags:
  - architecture
  - audit
  - calibration
  - interim
  - planning
related:
  - "06 Execution/reports/backed-depth-divergence-capture-2026-07-06-v2.md"
  - "06 Execution/reports/backed-depth-divergence-cohort-integrity-qa-2026-07-06-v1.md"
  - "06 Execution/reports/gate4-discrimination-sufficiency-criterion-2026-07-05-v1.md"
  - "06 Execution/reports/gate4-coverage-sport-supply-diagnostic-2026-07-05-v1.md"
  - "06 Execution/reports/highest-leverage-slice-discovery-interrogate-audit-2026-07-05-v1.md"
---

# Interim System Improvement Clinical Audit 2026-07-06 v1

Doctor-style diagnostic of DAI during the waiting window between the captured 2026-07-06
divergence cohort (QA passed, games now in progress) and its settlement (after official
finals, realistically 2026-07-07 morning). No-spend; audit + planning only.

## 1. objective

Identify the highest-value improvements executable during interim windows -- without
corrupting measurement, generating paid runs, tuning prematurely, or adding debt. Separate
urgent treatment from watchful waiting.

## 2. repo state (phase 0)

| item | reading | classification |
|---|---|---|
| dai branch/HEAD | main @ `d79c38f`, 0 ahead / 0 behind | expected, safe |
| dai dirty | `DevCore.Data.csproj` phantom only | expected, safe -- now DIAGNOSED (see labs: EOL phantom) |
| dai-vault branch/HEAD | main @ `b6cdce1`, **3 ahead / 0 behind** | expected; push pending approval |
| dai-vault unpushed | `b6cdce1` (QA), `aa8c82f` (capture v2), `7fef9e8` (readiness) | expected, safe; single-machine risk until pushed |
| dai-vault untracked | `06 Execution/system-state-synopsis-v1.md` | expected; leave untracked |
| services | DevCore.Api :5007 DOWN, agent-service :8000 DOWN, devcore-sql Exited | expected (operator-stopped after QA); safe |

No unexpected drift. No blockers.

## 3. current context

- 6-run backed_depth divergence cohort captured 2026-07-06 (~$0.0043), QA passed, all 6
  SingleMatch settlement-ready, 5 agree / 1 disagree (823036 MIL@STL the sole divergence).
- At audit time (15:03 ET) the cohort's games are IN PROGRESS (first pitch 14:10 ET); last
  games end ~00:30 ET -> settlement window opens tonight/2026-07-07 morning.
- Gate 4 (`discrimination_hybrid_v1`) FALSE on merits: `discrimination_inverted` (0.75
  region acc 0.619 vs 0.80 acc 0.533), `insufficient_market_disagreement` (4 < 10),
  `insufficient_market_coverage` (0.565 < 0.60).
- Next required evidence action: settlement-only reconciliation of the 6 runs.

## 4. system vitals (phase 2)

| vital | current reading | healthy range / expectation | status | evidence |
|---|---|---|---|---|
| AgentRuns | 279 | 273 + 6 captured | HEALTHY | SQL count, QA v1 (09:25 ET); services down since -- cannot have moved |
| Outcomes / Evaluations | 112 / 112 | unchanged until settlement | HEALTHY | SQL join-count, QA v1 |
| Captured cohort | 6/6 complete, active, no dups, registry backed_depth, no fallback | all of those | HEALTHY | QA v1 (durable PromptRouteProvenanceJson) |
| Settlement readiness | 6/6 SingleMatch "identity /reconcile is safe"; 0 outcomes/evals | ready, unambiguous | HEALTHY (waiting) | QA v1 reconcile-precheck x6 |
| Gate 4 | conclusionsAllowed = FALSE | FALSE until real evidence | HEALTHY-AS-DESIGNED | gate4 criterion report 07-05 |
| failingReasons | discrimination_inverted; disagreement 4<10; coverage 0.565<0.60 | expected pre-evidence | STABLE | same |
| Market disagreement (settled) | 4 (2c/2i) + 1 unsettled captured | >=10 readable needed | THIN (blocked on capture+settlement) | coverage diagnostic 07-05 + capture v2 |
| Market coverage | 0.565 (52/92); projected ~0.59 after settling 6 (58/98, all 6 baselined) | >= 0.60 | IMPROVING, still short | coverage diagnostic + projection |
| agent-service tests | **436 passed in 4.81s (rerun this slice)** | all green | HEALTHY | pytest this audit |
| DevCore.Api tests | 1052 passed (last verified 2026-07-05) | all green | UNKNOWN-FRESH (stale 1 day; rerun needs SQL up) | readiness v1 |
| Repo cleanliness | dai clean-except-phantom; phantom root-caused | clean | HEALTHY | phase 0 + labs |
| Unpushed commits | dai 0; dai-vault 3 | 0 (or intentional) | WATCH (single-machine loss risk) | git status -sb |
| Paid-call state | 6 calls today ($0.004259); 0 this slice | capped, logged | HEALTHY | capture v2 report |
| Registry default-off | canary absent from .env; agent-service down | default-off | HEALTHY | QA v1 + phase 0 |
| Buyer-claims status | none made; performance claims gated behind Gate 4/5 | none until evidence | HEALTHY | interrogate audit 07-05 |

## 5. symptoms (phase 2/3 observations)

1. Gate 4 still FALSE (three reasons) -- expected, but creates pressure to "do something."
2. Market disagreement sample thin: 4 settled + 1 captured -- the core evidence hole.
3. Cost evidence non-durable: `devcore.cost` logger -> stdout `basicConfig` only
  (main.py:27); per-run evidence survives only in session logs + committed reports.
4. DAI-vs-market agreement is derived, not persisted on the run: calibration `/rows`
  derives it (PromptRouteCalibrationExport.cs), but the artifact stores the numeric market
  baseline as a prose string (`SourceDepth.market_odds.Detail`).
5. Pre-settlement QA is manual: QA v1 was ad-hoc SQL + curl; every future cohort will need
  the same checks.
6. Capture slate screening is manual: capture v2 ran StatsAPI + odds + source-readiness by
  hand; repeatable but operator-intensive.
7. dai-vault 3 commits unpushed (single-machine risk); csproj EOL phantom recurs in every
  handoff as noise.
8. Interim windows invite overbuild -- the 07-05 interrogate audit already ruled: machinery
  is mature; do NOT build more protocol/Interrogate.

## 6. differential diagnosis (phase 4)

| symptom | possible cause | evidence for | evidence against | test to confirm | treatment |
|---|---|---|---|---|---|
| Gate 4 still false | (a) criterion broken; (b) evidence genuinely insufficient | (b): disagreement 4<10, coverage 0.565, inversion 0.619 vs 0.533 | (a) ruled out: criterion replaced 07-05, TDD-proven satisfiable on discriminating corpus | settle cohorts; watch reasons shrink | watchful waiting + capture/settle cycles; NO gate edits |
| Disagreement thin | (a) DAI is a market echo; (b) sample too small to tell | 07-04 cohort 7/7 agree | v2 close-favorite prefilter produced 1/6 divergence -> prefilter works | settle 823036; run more close-favorite captures | more divergence-prefiltered captures after settlement |
| Cost evidence non-durable | stdout-only logging (basicConfig, no file sink) | main.py:27; scratchpad log was the only per-run record | committed reports carry aggregates | inspect logging config (done) | Durable Cost Evidence v1: additive file/JSONL sink, observability-only |
| Agreement derived not persisted | artifact schema kept lean; read model owns derivation | OutputJson stores prose Detail string | /rows already derives marketAgreement correctly; settlement keys on gamePk only | read PromptRouteCalibrationExport.cs (done) | acceptable now; structured baseline field only in a future approved artifact-version slice (NOT mid-cohort) |
| WNBA doesn't advance Gate 4 | spread-only baseline; not backed_depth | 07-05 sport-supply audit (code-cited) | none | already confirmed | defer WNBA; revisit only for factory breadth after Gate 4 evidence flows |
| Buyer trust surface premature | no settled edge evidence exists | Gate 4 FALSE; 5 agree/1 disagree unsettled | internal-only packaging is allowed | settlement outcomes | defer; internal "evidence pending" framing only after this cohort settles |
| ProbeRefresh Stage-3 blocked | gated on Gate 4/5 by design | interrogate audit: IntegratedDormant, no live caller | none | none needed | correct gating -- do not touch |
| Interim overbuild pressure | idle window + mature machinery | this slice exists | prior audits consistently said "measure, don't build" | n/a | rank slices by settlement usefulness; reject busywork below the bar |

## 7. labs / evidence inspected (phase 1 + this slice)

- Reports read: capture v2, cohort QA v1, gate4 criterion handoff, criterion review handoff,
  coverage/sport-supply handoff, interrogate audit handoff, readiness v1.
- **pytest rerun this slice: 436 passed in 4.81s** (offline, no-spend).
- csproj phantom root-caused: `git ls-files --eol` -> `i/lf w/crlf`, `core.autocrlf=true`,
  **no `.gitattributes`** -> committed-LF/checked-out-CRLF phantom. Benign; empty textual
  diff; permanent fix = `.gitattributes` + renormalize commit in dai (deferred; real commit).
- `MarketAgreement` derivation confirmed present in
  `platform/dotnet/DevCore.Api/AgentRuns/PromptRouteCalibrationExport.cs`.
- Cost sink confirmed stdout-only: `services/agent-service/main.py:27` `logging.basicConfig`.
- Repo/service state: phase 0 table. No DB reads needed (QA v1 counts current; services down
  since, nothing could write).

## 8. contraindications (phase 5) -- do NOT do these now

| contraindicated action | why |
|---|---|
| Tune prompts / confidence generation | Gate 4 FALSE = no calibration evidence to tune against; would also invalidate comparability with the captured-but-unsettled cohort |
| Change Gate 4 again | criterion replaced 24h ago and TDD-proven; changing it again before any new evidence arrives is goalpost-moving by definition |
| Build buyer claims | zero settled edge evidence; 5/6 of the new cohort agrees with market -- nothing claimable exists |
| Enable ProbeRefresh Stage-3 mutation | explicitly gated on Gate 4/5; mutating artifacts now would corrupt the measurement chain |
| Add WNBA for volume | 07-05 audit: spread-only baseline, does not advance backed_depth Gate 4; volume without the right dimension is noise |
| More paid captures before settlement | the 6-run cohort is the instrument in flight; spend before reading it is uninformed spend |
| Reconcile before official finals | outcomes would be wrong/unverifiable; violates the settlement-only-after-finals contract |
| Backfill market baselines from current external data | not decision-time data -- would fabricate baselines (07-05: uncovered runs are old/unbackfillable; REJECTED) |
| Mid-cohort artifact schema change (e.g. structured baseline field) | would split the cohort across artifact shapes mid-measurement; do it, if ever, between cohorts with version bump |

## 9. candidate improvement areas (phase 3)

| candidate | category | classification |
|---|---|---|
| Pre-Settlement QA Script v1 (batch precheck + integrity manifest) | workflow reliability | **SAFE NOW** |
| Settlement Readiness Manifest Standard | measurement quality | SAFE NOW (fold into QA script output) |
| Durable Cost Evidence v1 (file/JSONL sink for devcore.cost) | observability | SAFE SOON (tiny additive runtime-adjacent change; own approved slice) |
| Capture Slate Generator v1 (StatsAPI+odds+readiness -> frozen slate doc) | workflow reliability | USEFUL, WAIT (needed before capture v3, after settlement) |
| Repo/Vault sync hygiene (push 3 commits; .gitattributes fix) | debt | SAFE with approval (push needs explicit approval; renormalize = real dai commit) |
| No-Paid-Call Guardrail Test v1 | runtime safety | MARGINAL (caps + default-off already enforced; test adds little) |
| Registry Canary Guardrail Test v1 | runtime safety | MARGINAL (canary config default-off already unit-tested; 436 suite green) |
| Market Agreement Persistence (artifact field) | measurement quality | PREMATURE (mid-cohort schema change contraindicated; /rows already derives it) |
| Tool Gateway Durable Invocation Audit v1 | observability | VERIFY-FIRST (07-05 audit says authz+audit already on every run; durability unconfirmed -- check before building) |
| WNBA Feasibility Boundaries v1 | factory architecture | ALREADY ANSWERED (07-05: CaptureReadyWithSmallConfig, spread-only; boundary = enable only for breadth, never for Gate 4) |
| Buyer Trust Internal Packaging v1 | buyer trust readiness | PREMATURE (defer until first settled divergence evidence) |
| More Interrogate/protocol machinery | factory architecture | NOT WORTH IT (ruled by 07-05 audit: mature + correctly gated) |

## 10. ranked interim slices (phase 6)

Scoring 1-5: measurement integrity / risk reduction / factory leverage / implementation
readiness / cost discipline / settlement usefulness / revenue relevance.

| rank | slice | score | notes |
|---|---|---|---|
| 1 | **Pre-Settlement QA Script v1** (subsumes batch reconcile-precheck report + manifest standard) | 4/4/5/5/5/5/2 = **30** | dev-tooling only (pattern: run-artifact-calibration.ps1); 0 runtime change, 0 DB writes, 0 paid; makes tonight's settlement mechanical and every future cohort's QA repeatable; codifies QA v1's manual checks |
| 2 | Settlement Readiness Manifest Standard v1 | 3/3/4/5/5/4/2 = 26 | folded into #1 as its markdown output; not a standalone slice |
| 3 | Durable Cost Evidence v1 | 4/4/3/4/5/2/2 = 24 | additive JSONL file sink for `devcore.cost`; observability-only but touches agent-service logging -> own small approved slice, after settlement |
| 4 | Capture Slate Generator v1 | 3/3/5/4/4/1/3 = 23 | high factory leverage for capture v3+; zero settlement value; sequence after settlement |
| 5 | Repo/Vault Sync Hygiene v1 | 2/4/2/5/5/2/1 = 21 | push 3 vault commits (approval-gated) + `.gitattributes` renormalize (kills the phantom noise permanently); tiny |
| 6 | No-Paid-Call Guardrail Test v1 | 3/4/2/3/5/1/1 = 19 | marginal: caps enforced operationally; a CI assertion adds thin safety |
| 7 | Registry Canary Guardrail Test v1 | 3/3/2/4/5/1/1 = 19 | canary default-off already unit-tested; marginal |

Below the bar: Buyer Trust Internal Packaging (premature), Market Agreement Persistence
(contraindicated mid-cohort), Tool Gateway Audit (verify-first), WNBA (answered), more
Interrogate machinery (rejected by prior audit).

## 11. recommended interim slice (phase 7)

**Chosen: Pre-Settlement QA Script v1** -- `scripts/dev/sports/preflight-settlement.ps1`
(PS7 dev tooling, same family as run-artifact-calibration.ps1). Given a list of gamePks:
re-verify membership/status/exclusion from read-only SQL or `/rows`, pull durable
provenance (registry/fallback/regime/attribution), run `/reconcile-precheck` per game,
confirm 0 outcomes/evaluations, and emit a settlement-readiness manifest markdown to the
vault. Exactly what QA v1 did by hand, made repeatable.

- why chosen: highest settlement usefulness (the next event is settlement, tonight or
  tomorrow morning); zero runtime/DB/paid footprint; converts a proven manual procedure into
  a factory station; reusable for every future cohort.
- expected code changes: 1 new dev script (+ optional docs). No app code.
- expected DB writes: 0. expected paid calls: 0.
- risk: near-zero (read-only script; worst case it reports wrong and the human re-checks).

## 12. runner-up

**Durable Cost Evidence v1** -- additive JSONL file sink for the `devcore.cost` logger
(rotating file under a logs dir, .gitignored), so per-run cost lines survive the session
without depending on scratchpad capture. Why second: real observability gap (confirmed
stdout-only at main.py:27), but it touches agent-service runtime config (however additively),
so it belongs in its own small approved slice -- and it has zero settlement usefulness for
tonight. Run it after settlement.

- what would change the recommendation: if official finals are already posted when the
  operator returns, skip the interim slice entirely and run settlement first -- settlement
  outranks all interim work. The QA script can be built in the next waiting window.

## 13. deferred work

- Capture Slate Generator v1 (before capture v3).
- Durable Cost Evidence v1 (after settlement).
- `.gitattributes` + renormalize (next natural dai commit window).
- Vault push (on explicit approval; 3 commits pending).
- Structured market-baseline artifact field (between cohorts, version-bumped, approved).
- Buyer trust internal packaging (after first settled divergence evidence).
- WNBA enablement (only if factory breadth becomes the goal).
- Tool Gateway audit durability check (fold into a future observability slice).

## 14. validation proof

- 0 paid calls, 0 model calls, 0 AgentRuns, 0 outcomes/evaluations, 0 DB writes (services
  down throughout; no DB connection opened this slice).
- pytest: 436 passed (offline).
- dai `git status`: only the csproj phantom; HEAD `d79c38f` unchanged.
- All numeric vitals sourced from this session's QA (same-day) or 07-05 reports; none
  fabricated; DevCore.Api test freshness explicitly marked stale-1-day.

## 15. what did not change

runtime code, prompts, registry recipes, routing, confidence generation, Gate 4 logic,
buyer copy, schema/migrations, ProbeRefresh posture, captured artifacts: unchanged.
agent-service never started. No pushes. Docs-only.
