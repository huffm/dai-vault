---
title: "HANDOFF: Market Attribution Fidelity Debug + Remediation Plan v1 -- DONE (2026-07-07)"
type: "handoff"
date: "2026-07-07"
status: "COMPLETE -- mutation point located; guard-first hybrid remediation chosen; implementation prompt ready"
project: "DAI"
slice: "Market Attribution Fidelity Debug + Remediation Plan v1"
related:
  - "06 Execution/reports/market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md"
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
---

# HANDOFF: market attribution fidelity debug + remediation plan v1 (2026-07-07)

## 1. objective

trace exactly where staged market truth becomes wrong artifact prose (fixtures 823036,
824820) and choose the minimal safe remediation before any implementation.

## 2. outcome

COMPLETE. report:
`06 Execution/reports/market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md`.
**mutation point = model generation (step 5), enabled by two prompt-context design
hazards found in source:** (1) the market overlay injects the consensus as a bare side
label -- `moneyline consensus favors the {consensus_side} side.`
(mlb.overlay.market.backed_depth.v1.txt line 5; sports_analyzer.py ~896) -- the TEAM is
never named, so the model resolves "the away side" itself mid-generation and lands on
its lean's team; (2) the numeric context is the HOME-only RAW per-side median (not
de-vigged: 823036 persisted home 0.500 + away 0.537, sum > 1) rounded whole
(format_pct_whole), so both bad fixtures rendered "50%" beside "consensus favors the
away side". corpus correlation: incoherent-looking blocks misattributed 5/7 (71%) vs
3/30 (10%) coherent -- aggravator, not sole cause (824500 @47 / 824662 @40 / 824012 @43
flipped with coherent numbers). all persisted steps (retrieval, staging, derivation,
prompt values, parse, read model) carry CORRECT data. **remediation: Option E staged
hybrid -- deterministic Market Attribution Fidelity Guard v1 FIRST (no generation
change; paste-ready implementation prompt in report section 10, full test plan section
9), then Prompt Market Context Hardening v1 as a separate approval-gated paid-canary
slice measured against the guard's baseline.** taxonomy correction: 822956's "-1.5" was
NOT hallucinated (the prompt's run line block carries real spreads); market-TYPE
fidelity is withdrawn as a defect class.

## 3. repo state before / after

- dai: main @ `98b3306`, 0/0, phantom csproj only. UNCHANGED (read-only slice).
- dai-vault before: `797d3a1`, 0/0, known untracked noise. after: report + this handoff
  committed and pushed (sha in session closeout).

## 4. services used

DevCore.Api :5007 read-only (/rows medians pulls x2); artifact JSONs reused from the
session scratchpad cache. agent-service NOT started; no statsapi; no external calls;
0 paid calls.

## 5. work performed

- skills gate; superpowers:systematic-debugging loaded and followed (phase 1 evidence
  before any remediation talk; component-boundary tracing).
- phase 0 baselines clean (dai `98b3306`, vault `797d3a1`).
- phase 1 trace: 10-step source-to-prose table for both fixtures; every persisted step
  verified correct against /rows + artifacts + code.
- phase 2 mutation isolation: read mlb.overlay.market.backed_depth.v1.txt,
  sports_analyzer.py market injection (~882-913) + format_pct_whole (319), the
  no-fabrication instructions (220-255), OddsMarketClient BaseballMarketContext mapping
  (218-230); pulled persisted medians for fixtures + controls; ran the corpus
  coherence-vs-misattribution correlation (37 rows with medians: 5/7 vs 3/30).
- phase 3-4: five remediation options evaluated (D eliminated by evidence; C rejected
  as new integrity risk; E chosen: A guard first, B prompt hardening second).
- phase 5-6: exact test plan (6 fixtures + 6 invariants) + paste-ready guard
  implementation prompt.
- phases 8-10: validation, vault-only commit, push.

## 6. files changed

dai: none.
dai-vault (one commit):
- `06 Execution/reports/market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md` (new)
- `06 Execution/reports/market-attribution-fidelity-debug-handoff-2026-07-07-v1.md` (this file)

## 7. db writes / side effects

0 db writes; 2026-07-07 cohort untouched (still unreconciled; readiness guard governs);
0 external calls.

## 8. paid calls / cost

0 paid model calls, $0.00.

## 9. validation proof

- every trace claim cites a file/line (template line 5; sports_analyzer.py 319/896/900;
  OddsMarketClient 224-228) or a live /rows value (823036 medians 0.500/0.537; 824820
  0.505/0.535; controls 54%).
- correlation computed from the live corpus (command output in session): incoherent 7
  rows -> 5 mismatches; coherent 30 -> 3.
- no code/prompt changed (dai git status = phantom only); no db writes; no paid calls;
  cadence not resumed; accidental divergences never called candidate edge.

## 10. what did not change

all code, prompts, templates, gates, confidence, models, registry flags, buyer copy,
database contents, the unsettled 07-07 cohort, the capture pause.

## 11. open issues

- guard implementation pending (prompt ready in report section 10); capture stays PAUSED
  until it is live.
- prompt hardening slice will need registry recipe reversioning + paid canary validation
  -- it changes generation behavior and owns the byte-equality invariant transition.
- 824662 duplicate-active hazard still open (unchanged, from the taxonomy slice).
- the 07-07 cohort settles first if check-settlement-finals.ps1 returns READY
  (~00:50 ET 2026-07-08).

## 12. recommended next slice

**Market Attribution Fidelity Guard v1** (implementation, TDD; paste-ready prompt in the
report section 10) -- unless the 07-07 cohort reaches READY first, in which case the
taxonomy-aware settlement runs first with the accidental label preserved. after the
guard: Prompt Market Context Hardening v1 (separate, approval-gated).

## 13. suggested next prompt

verbatim from market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md
section 10.
