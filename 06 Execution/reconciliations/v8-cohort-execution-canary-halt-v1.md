---
title: "v8 Cohort Execution -- Canary Halt v1"
type: "reconciliation"
date: "2026-07-03"
status: "halted-at-canary (operator go/no-go pending)"
project: "DAI"
slice: "v8 Targeted Measurement Cohort Execution"
repos:
  dai: "unchanged (beed3fc)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - canary
  - halt
related:
  - "06 Execution/plans/v8-targeted-measurement-cohort-plan-v1.md"
  - "02 Platform/decisions/0006-canonical-reconciliation-residue-contract.md"
---

# v8 Cohort Execution -- Canary Halt v1

**slice:** execute the approved v8 backed_depth measurement cohort (10 games, cap 10 gpt-4o-mini calls),
canary-first. **outcome: HALTED after 1 paid call on the operator's own canary stop-condition.** 9 calls
unspent. no settlement (games are 07-04, not final).

## services + preflight (all clean)

Docker/devcore-sql up; DevCore.Api :5007 /health ok (HEAD beed3fc, Development, dev bypass);
agent-service :8000 started this slice (uvicorn, OPENAI key loaded). all 10 approved gamePks preflighted
via StatsAPI: **Scheduled, both probables announced, none postponed/scratched** -- all keep. all 10 mapped
exactly to platform `/upcoming` team-name inputs for 2026-07-04.

## canary #1 (paid call 1/10)

generated SD@LAD via `POST /api/agent-runs {mlb, Los Angeles Dodgers, San Diego Padres, 2026-07-04}`.
- **AgentRunId:** be49433e-f36b-1410-8173-00373db4b724
- **identity:** SourceProvider mlb_statsapi, **ExternalGameId 823932** (matches approved gamePk exactly).
- **market capture WORKS (07-02 failure did NOT recur):** sourceDepth `market_odds: enriched -- multi-book
  moneyline depth: 3 books`; /rows `marketBookCount=3, marketConsensusSide=home, marketMedianHomeImpliedProb
  0.737, marketAgreement=true`. starter `enriched`.
- **decision:** leanSide home (LAD), **confidence 0.80**, advertisedStrength High, evidenceRichness 2.
- **BUT route label absent:** `selectedDataRegime = None`, `promptSource = live`,
  `registryAuthoritativeEnabled = False`, `regimeAllowlisted = False`, `fallbackReason = disabled`,
  **promptRouteKey = "unknown"**.

## why (root cause -- config, not defect)

The Registry-Authoritative Prompt Canary is **DEFAULT-OFF** (`services/agent-service/app/services/
registry_prompt_canary.py`: "a DEFAULT-OFF, allowlist-restricted canary"). Its enable flag/allowlist env is
**absent from `.env`**, so runs take the live path and NO `selectedDataRegime` is stamped -> pooled
attribution lands them in route **`unknown`** (already n=68), NOT `enriched_market_backed_depth`. The
historical 16 enriched_backed_depth + 3 enriched_market_missing rows were generated in a PRIOR window when
registry routing was enabled; it is off now.

## canary verdict = STOP (operator's own condition)

The operator's approval said: "stop if selectedDataRegime is not enriched_market_backed_depth." It is
`None`. **Halted at 1/10 calls; 9 unspent.** Producing the regime label would require enabling
registry-authoritative routing = a **prompt-selection behavior change**, which is a hard non-negotiable for
this slice. So the backed_depth ROUTE evidence cannot be produced under the approved (unchanged) pipeline.

## what the canary DID and did NOT clear

- **CLEARED:** the primary 07-02 risk (market_missing) -- market capture produces multi-book backed depth;
  the run is genuinely enriched-starter + backed-depth-market by DATA (auditable via sourceDepth + /rows
  market fields), at **confidence 0.80** (the target bucket) with correct identity.
- **NOT CLEARED:** route attribution -- runs land `unknown`, not `enriched_market_backed_depth`, because
  regime routing is config-off and forbidden to toggle here.

## go/no-go for the operator (2 options)

- **(A) CONTINUE the remaining 9 (needs re-approval -- deliverable changed).** You still get ~9 runs at
  ~confidence 0.80 -> the **0.80 confidence bucket (12 -> ~18-21) very likely crosses the n>=15 gate**, which
  is the ACTUAL blocking gate on conclusionsAllowed, PLUS market-agreement + home/away evidence, all with
  data-level backed_depth auditable on /rows. Trade-off: these grow the `unknown` route, NOT the
  `enriched_market_backed_depth` route line. Confidence-bucket value is route-independent, so the primary
  gate still moves.
- **(B) HALT and defer.** If the `enriched_market_backed_depth` ROUTE line specifically must grow, that
  needs a separate, explicitly-approved slice to enable registry-authoritative routing (a prompt-selection
  change) BEFORE spending -- then re-run this cohort.

**Recommendation:** (A) with your explicit re-approval. The confidence-bucket gate is the real blocker on
`conclusionsAllowed`, it is route-independent, and this cohort can move it now; route attribution can be
recovered later via a registry-routing slice. I did not spend the 9 without your call, per the canary gate.

## settlement watch plan (regardless of A/B)

canary run be49433e (SD@LAD 823932) is generated + unsettled. 07-04 games go Final ~2026-07-04 21:35Z
through 2026-07-05 05:40Z. settle after all-final (~2026-07-05 06:00Z) via the canonical residue contract
(precheck -> /reconcile, source=statsapi_final, sourceRef="gamePk <pk>", notes with final score + writer
path), then rerun pooled reassessment + residue analysis. do NOT settle before finality.

## safety ledger

paid calls **1** (canary; cap 10, 9 unspent); new game runs **1** (be49433e, approved SD@LAD); reconciliation
writes 0 (not final); DB migrations 0; prompt/prompt-selection/decision/confidence/buyer/denominator changes
NONE; no backfill; no CaptureMode; no unapproved games generated. no dai code changed.
