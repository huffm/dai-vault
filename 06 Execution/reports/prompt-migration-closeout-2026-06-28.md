---
title: "Prompt-Registry Migration -- Nightly Closeout 2026-06-28"
type: "evidence-report"
date: "2026-06-28"
status: "complete"
project: "DAI"
slice: "Prompt-Registry Migration Nightly Closeout 2026-06-28"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - provenance
related:
  - "06 Execution/reports/phase-3-2-global-prompt-routing-hardening-v1.md"
---

# Prompt-Registry Migration -- Nightly Closeout 2026-06-28

**status:** state reconciliation (docs only; no code/model/paid calls this slice)
**date:** 2026-06-28
**type:** nightly closeout for the 2026-06-28 prompt-registry migration work.

## current repo state

| repo | HEAD | upstream | tree |
|---|---|---|---|
| `dai` | `f3e431e` | synced with origin/main (0 ahead) | clean |
| `dai-vault` | `6f48ee7` | synced with origin/main (0 ahead) | clean except the intentional untracked `06 Execution/system-state-synopsis-v1.md` |

**Pushed/unpushed:** everything is PUSHED. The three docs commits the latest handoff called "unpushed" (`fd28e8d`,
`caa32ab`, `6f48ee7`) were pushed at the operator's "Push" after that handoff was written; both repos are now 0 ahead.
`dai` code has been unchanged since `f3e431e` (all evidence slices were docs-only / operational).

## services / env posture

- agent-service :8000 -- DEFAULT (canary OFF, capture OFF; no `DAI_MLB_*` env in the running process).
- platform-api :5007 -- up (Development).
- devcore-sql -- up (docker).
- All capture/soak/log/response JSONL are scratch / out-of-repo; NONE committed.

## prompt migration state map (the slice chain)

Phase 4 provenance + manifest integrity -> 4.1 persistence + CI -> Migration Readiness -> Shadow-Parallel Routing
(default-off) -> Cohort Soak (representative GO) -> Live-Data Shadow Soak (readiness; was access-blocked) -> Request
Input Export -> Analyzer Request Capture (first real capture) -> Real-Cohort Live Soak (15-game real GO) ->
Registry-Authoritative Prompt Canary (default-off, equality-gated, allowlist=2) -> Canary Real Confirmation (regime 1)
-> Canary Backed-Depth Confirmation (regime 2) -> Multi-Slate Regime Coverage (matrix) -> Starter-Missing Regime
Capture (regimes 3+4 real-soak-clean). Mechanics today: the registry prompt can become model input ONLY when proven
byte-identical to the live prompt, for allowlisted regimes only; the live builder is always the authority and floor.

## real-soak evidence by regime (4 of 9 have real evidence)

| regime | real-soak-clean | canary runtime-confirmed | allowlist |
|---|---|---|---|
| starter_enriched_market_missing | YES | YES (Dodgers@Padres) | ALLOWLISTED |
| starter_enriched_market_backed_depth | YES | YES (Yankees@Red Sox) | ALLOWLISTED |
| starter_missing_market_missing | YES (2 real inputs) | no | off |
| starter_missing_market_backed_depth | YES (2 real inputs) | no | off |
| starter_enriched_market_backed (single-book) | no (representative-only) | no | off |
| starter_named_market_missing | no (representative-only) | no | off |
| starter_named_market_backed | no (representative-only) | no | off |
| starter_named_market_backed_depth | no (representative-only) | no | off |
| starter_missing_market_backed (single-book) | no (representative-only) | no | off |

## canary-authoritative evidence by regime

Runtime-confirmed (registry prompt selected as model input, byte-identical, one call, no mismatch): the two enriched
regimes only. See [[registry-canary-real-confirmation-v1]], [[registry-canary-backed-depth-confirmation-v1]].

## current allowlist

`DEFAULT_ALLOWLIST` (code) = exactly two regimes: `starter_enriched_market_missing`,
`starter_enriched_market_backed_depth`. UNCHANGED today. The two new real-soak-clean starter_missing regimes are NOT
allowlisted.

## remaining unproven regimes (5, representative-only)

`starter_enriched_market_backed` (single-book), `starter_named_market_missing`, `starter_named_market_backed`,
`starter_named_market_backed_depth`, `starter_missing_market_backed` (single-book). All involve the `named` starter
state or a single-book `backed` market -- neither observed in real data across discovered slates (6/28-7/1); may be
genuinely rare.

## paid-call summary for the day (2026-06-28)

| slice | paid model calls |
|---|---|
| Live-Data Shadow Soak v1 | 0 (access-blocked; readiness only) |
| MLB Request Input Export v1 | 0 |
| MLB Analyzer Request Capture v1 | 1 (Yankees@Red Sox first capture) |
| Real-Cohort Live Soak v1 | 14 (rest of the 15-game slate; operator-approved) |
| Registry Canary Real Confirmation v1 | 1 (Dodgers@Padres; operator-approved) |
| Registry Canary Backed-Depth Confirmation v1 | 1 (Yankees@Red Sox; operator-approved) |
| Multi-Slate Regime Coverage v1 | 0 |
| Starter-Missing Regime Capture v1 | 4 (operator-approved) |
| **total** | **21** |

Each was a normal single live analyze call (no second call per run). All multi-call cohorts were explicitly
operator-approved; the sandbox blocked unapproved batches until approval was given.

## generated artifact policy

All capture/request/soak/provenance/response/log JSONL live under the scratch directory (out of both repos) and are
NEVER committed. Only vault docs (evidence reports + handoff entries) are committed. The 9 prompting overlay templates +
`manifest.json` are unchanged.

## docs map (prompting product doc set -- state pointers, not rewritten)

Foundations: [[controlled-dynamic-prompt-assembly-architecture-v1]], [[prompt-registry-contract-v1]],
[[prompt-assembly-engine-v1]], [[data-regime-overlay-mlb-prompt-equivalence-v1]],
[[market-depth-starter-named-overlays-v1]]. Provenance/integrity:
[[prompt-provenance-and-manifest-integrity-v1]], [[prompt-provenance-persistence-v1]]. Migration path:
[[live-migration-readiness-v1]], [[live-prompt-routing-shadow-parallel-v1]], [[shadow-parallel-cohort-soak-v1]],
[[live-data-shadow-soak-v1]], [[mlb-request-input-export-v1]], [[mlb-analyzer-request-capture-v1]],
[[real-cohort-live-soak-v1]]. Canary + coverage: [[registry-authoritative-prompt-canary-v1]],
[[registry-canary-real-confirmation-v1]], [[registry-canary-backed-depth-confirmation-v1]],
[[multi-slate-regime-coverage-v1]], [[starter-missing-regime-capture-v1]]. (No index file exists; this map is the
pointer. A dedicated `00-prompting-index` could be created later if the set keeps growing -- not done tonight to avoid
churn.)

## remaining risks

- **5 regimes representative-only.** No real evidence; `named` and single-book `backed` may be hard to source.
- **Modest samples.** 1-2 real inputs per non-enriched regime; the enriched regimes have the largest real samples.
- **Allowlist is 2; 4 regimes are real-soak-clean.** The gap between "real-soak-clean" and "canary-authoritative
  allowlisted" is deliberate -- widening is a separate decision.
- **Single-writer capture.** Sequential runs avoided torn JSONL; concurrent capture remains out of scope.
- **Operational dependency.** Real evidence needs platform-api + devcore-sql + odds availability; future-dated games
  often lack markets (-> market_missing).

## next recommended slice

**Starter-Missing Canary Confirmation v1** (operator-approved, paid): runtime canary confirmation for
`starter_missing_market_missing` and `starter_missing_market_backed_depth`. Mechanism: set
`DAI_MLB_REGISTRY_PROMPT_CANARY_REGIMES` to those two regimes for the confirmation run ONLY (a runtime env override) --
do NOT change `DEFAULT_ALLOWLIST` in code in that slice unless explicitly instructed. Confirm registry-selected +
byte-identical + one call + no mismatch per regime, then a SEPARATE slice may widen the code allowlist to the four
runtime-confirmed regimes.

## do-not-do list for tomorrow

- Do NOT widen `DEFAULT_ALLOWLIST` in the same slice as the starter_missing canary confirmation (unless explicitly
  instructed).
- Do NOT enable broad/all-regime registry-authoritative routing.
- Do NOT add the `named` or single-book `backed` regimes to anything without real-soak evidence.
- Do NOT run paid calls without explicit operator approval (number/slate/games).
- Do NOT commit scratch capture/soak artifacts.
- Do NOT change prompt wording, templates, manifest, model, temperature, confidence, or artifact copy.
- Do NOT harvest DB credentials; use the sanctioned API.
- Do NOT leave the agent-service in a non-default (capture/canary-on) posture after a run.

## reflections

- **What worked.** Evidence-gated, reversible slices; three independent default-off sidecars (capture, shadow,
  canary) that never alter live behavior when disabled; the strict byte-equality guard making "registry as model
  input" provably safe; deterministic offline reproduction of `select_model_prompt` as canary-success evidence (since
  success logs nothing); honest "unavailable" marking instead of fabricating regimes; the per-cohort operator approval
  gate for paid spend.
- **What created risk.** The canary (registry prompt reaching the model) was the highest-stakes change -- contained by
  a 10-path adversarial verification + the equality guard. Reusing scratch capture paths risked overwrite (mitigated by
  per-run fresh paths). Single-writer capture under concurrency (mitigated by sequential runs).
- **What should become skill doctrine.** (1) The default-off sidecar pattern: env config + pure builder + guarded
  wiring + "never break the live request." (2) "Real-soak-clean is necessary but not sufficient for the canary
  allowlist; allowlisting is a separate, evidence-cited decision." (3) Deterministic reproduction as runtime evidence
  when the success path is intentionally silent.
- **What to avoid in future migrations.** Fabricating unobserved regimes; coupling evidence capture with allowlist
  widening in one slice; assuming a regime is available without statsapi discovery; committing generated artifacts.

## related docs

- All prompting docs above; [[system-state-synopsis-v1]] (earlier full-system map; untracked by design);
  `06 Execution/handoffs/current-slice.md` (running slice log).
