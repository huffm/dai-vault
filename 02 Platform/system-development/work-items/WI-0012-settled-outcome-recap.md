---
title: "WI-0012 Settled Outcome Recap v1"
type: "plan"
date: "2026-07-14"
status: "complete"
project: "DAI"
slice: "WI-0012 Settled Outcome Recap v1"
repos:
  dai: "code (recap projection + export + endpoints; branch wi/0012-settled-outcome-recap)"
  dai-vault: "docs (this WI, MOC, current-slice, handoff at close)"
tags:
  - system-development
  - work-item
  - product
  - buyer
related:
  - "02 Platform/system-development/work-items/WI-0011-buyer-decision-brief-contract.md"
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
  - "06 Execution/plans/v1-release-critical-path-2026-07-14-v1.md"
---

# wi-0012 settled outcome recap v1

**Slice type:** production behavior change (new buyer-facing postgame contract + export).
**Opened:** 2026-07-14. Second implementation item of the frozen V1 critical path.

## problem

The product promise includes "settled outcome after completion", but no buyer surface
carries a settled result: the analyzer renders the pregame read only, history is mocked,
and GET /{id}/evaluation is an internal read no buyer path consumes. The concierge
workflow has no deliverable postgame recap.

## decision (governing product + architecture)

- The backend owns ONE canonical settled-recap projection and its deterministic Markdown
  export; the JSON endpoint and the export share it. No parallel recap semantics in
  Angular (no production Angular change required this slice).
- The "original read" portion EMBEDS the canonical WI-0011 buyer decision brief
  projection -- stance, lean/no-position, evidence-gated band, market context, buyer-safe
  prose, and identity are never re-derived.
- Recap state is a closed, explicit vocabulary: not_settled / settled_evaluated /
  settled_not_evaluated / no_position / excluded / inconsistent. Fail-closed: impossible
  residue (evaluation without outcome) returns `inconsistent` with a diagnostic warning,
  never an invented buyer result.
- The persisted evaluation is the sole source of correctness; the recap never recomputes
  it. No-position runs are never scored. Excluded runs render exactly "No result --
  event not evaluated." Per-read only: no aggregate record, win rate, or performance
  statistic anywhere on the contract.
- Identity (incl. requested + resolved gamePk, generated-at, settled-at) comes from
  persisted run and outcome data only.

## scope

1. `BuyerSettledRecap` projection (server-side, tenant-scoped, pure) embedding the
   WI-0011 brief; outcome block (final scores, winning team, settled-at) and evaluation
   block (persisted result + concise buyer-safe explanation) when present.
2. Fail-closed state rules per the authorization (unsettled / settled+evaluated /
   settled-not-evaluated / no-position / excluded / inconsistent).
3. `GET /api/agent-runs/{id}/recap` (JSON) + `GET /api/agent-runs/{id}/recap/markdown`
   (text/markdown), same auth/404 semantics as /brief; single coherent loader; no
   external calls; no writes.
4. Deterministic invariant-culture Markdown export from the same projection; claim-safe;
   no numeric confidence, aggregates, internal identifiers, or reconciliation mechanics.

## non-goals / exclusions

Settlement/reconciliation writes; outcome-matching or evaluation-calculation changes;
aggregate/accuracy surfaces; real history; WI-0013 (duplicate guard, metering); email
delivery; Stripe; deployment; auth changes; prompt/model/scoring/confidence-value/
routing/calibration changes; new signals; doubleheader ops; identity-status; WI-0002/
0003; DB migrations; push/merge (integration separately gated).

## acceptance criteria

The 15 criteria of the 2026-07-14 authorization, verbatim: canonical projection on
/recap; markdown shares it; WI-0011 semantics reused; persisted score+result rendered;
no-position never scored; unsettled never implies a result; excluded renders exact
non-evaluation copy; missing evaluation not recomputed; inconsistent residue fails
closed with an observable warning; identity persisted-only; no numeric confidence in
JSON or Markdown; no aggregate fields; deterministic output; claim-safety on all prose;
no locked-layer behavior change.

## required fixtures

Settled correct; settled incorrect; settled no-position; unsettled (no outcome);
excluded (823357 shape); outcome-without-evaluation; evaluation-without-outcome
(inconsistent); requested+resolved gamePk; legacy no-requested-pk; markdown determinism;
claim-unsafe prose suppressed; no numeric confidence; no aggregate language; tenant
isolation 404; WI-0011 reuse (not re-derivation).

## validation record (2026-07-14)

- red-first throughout: recap types (compile red -> 27 unit+integration green) and the
  review-driven state fixes (compile red on the new `no_result` state -> green).
- suites: DevCore.Api.Tests **1212/1212** (baseline 1176, +36: 27 initial recap tests + 9
  review-driven); sports-app vitest **134/134** (regression only -- no Angular production
  change); bundle compiles. All WI-0011 brief regressions green through the consolidated
  loader.
- live GET-only verification on real persisted runs (devcore-sql + DevCore.Api from the
  branch; runtime returned to cold; zero writes): 609d433e/823845 -> settled_evaluated,
  "Correct", final 3-2, winner Cleveland Guardians, settled-at from persisted ResolvedUtc,
  byte-identical text/markdown with no confidence/%/run-id; 6c9d433e/823357 ->
  excluded, "No result — event not evaluated.", no outcome surfaced; real lean-null run
  6816433e -> no_position with score 6-16 shown but explicitly not scored; real
  outcome-less run 0c8e433e -> not_settled. (settled_not_evaluated + inconsistent proven
  by integration fixtures -- no such residue exists in the real corpus, correctly.)
- focused review (3 angles): 10 findings -- 9 FIXED: honest `no_result` state for
  non-final outcome statuses (was a lying "settled, evaluation pending"); explicit
  eval-status switch (was keyed on copy length); disambiguated fail-closed warnings;
  partial-score residue omitted consistently from json AND markdown; suppression
  tripwire now fires on all four buyer surfaces via ONE consolidated loader (brief
  endpoints serve recap.Read -- the ~55-line loader duplication is gone); shared
  identity-header fragment kills renderer drift; winning-team side-word fallback for
  identity-less legacy rows; TryLabel empty-not-null contract. 1 REFUTED: WinningTeam
  correctly derives from the persisted OutcomeStatus (the outcome row owns the factual
  result; the evaluation owns only the verdict).
- accepted (documented): the read-block tense divergence between brief ("does not") and
  recap ("did not") is intentional pregame/postgame copy, not drift.

## links

- work item: WI-0012
- branch: `wi/0012-settled-outcome-recap` (dai, from `140b5a2`)
- pr: -- (not authorized)
- commits: dai `7152818` (5 files, +989/-43; implementation + tests + review fixes,
  LOCAL ONLY) + dai-vault docs commit at close (this WI, MOC, current-slice, handoff)
- tests: DevCore.Api.Tests 1212/1212; sports-app vitest 134/134
- verification notes: validation record above; live checks on 609d433e / 6c9d433e /
  6816433e / 0c8e433e
- docs updated: this WI; MOC; current-slice; handoff

## final disposition

Implementation complete, local only (2026-07-14). Review resolved (9 findings fixed,
1 refuted). Integration and push separately gated.
