# Run Eligibility and Supersession Contract v1

**date:** 2026-06-15
**status:** implemented. narrow backend change + one EF migration + tests in `dai`; report/ledger in `dai-vault`. no model spend, no new runs, no reconciliation, no outcomes/evaluations written, no analyzer/prompt/matcher-semantics/confidence/buyer change.
**classification:** lifecycle/eligibility contract enabling safe duplicate runs.

## Problem statement

One real-world game can have many analysis runs, but the canonical match key is `(SourceProvider, ExternalGameId)`. Rerunning a game (e.g. to replace a null-lean MLB run) creates a second AgentRun with the same provider key. With the matcher selecting on game identity + tenant alone, two active runs for one game produce `MultipleMatches`, which writes nothing and blocks reconciliation. A lifecycle/eligibility contract is needed before any rerun so stale duplicates are ignored without deleting history.

## Existing-capability audit (before implementing)

- `AgentRun.Status` is execution state (pending/completed/failed) only -- not match eligibility.
- No soft-delete, archived, inactive, calibration-eligibility, superseded, or exclusion field existed.
- `OutcomeReconciliationService.MatchAsync` filtered **only** `TenantKey + SourceProvider + ExternalGameId`; null-identity rows fall out by equality. No eligibility filter.
- Per-run reconciliation (`POST /api/agent-runs/{id}/outcome`) is a separate explicit path keyed by run id (not the matcher).
- Today, two active runs sharing the key -> `MultipleMatches` (writes nothing). Confirmed: no existing concept solved this.

## Concept model

- **GameIdentity** = `(SourceProvider, ExternalGameId)` (+ supporting ScheduledStartUtc/Season/team refs). One real game.
- **AnalysisRun** = one AgentRun. A game may have many. Each carries its own artifact/trace and an eligibility flag.
- **Outcome** = one settled result per game, recorded against the matched run.
- **Evaluation** = one deterministic lean-vs-outcome verdict per eligible matched run.

Doctrine: one game, many runs, one final outcome; only **active** runs participate in automatic matching and calibration.

## Chosen eligibility fields (minimal)

Two nullable columns on `AgentRun` (migration `20260615191124_AddRunMatchEligibility`, additive, clean Down):

- `ExclusionReason` (`nvarchar(32)`, null = **active/eligible**). When set, exactly one of `RunExclusionReasons`: `superseded` / `excluded` / `diagnostic` / `invalid`.
- `SupersededByAgentRunId` (`uniqueidentifier`, nullable) -- provenance link, meaningful only when reason is `superseded`; cleared for other reasons. Not an enforced FK (may reference a run created later).

Chosen over a non-null `RunStatus` enum because nullable columns need no data backfill: all existing rows are null = active automatically (verified: 129/129 active after migration). Null `LeanSide` does **not** imply exclusion -- a non-directional run stays active until explicitly superseded/excluded.

## Matcher behavior (after change)

`OutcomeReconciliationService.MatchAsync` now also filters `r.ExclusionReason == null`. The pure `OutcomeReconciliationMatcher` is unchanged (it still classifies whatever candidate set it is given). Resulting behavior:

- one active run shares the key -> `SingleMatch` (unchanged).
- one active + one superseded/excluded run share the key -> `SingleMatch` on the active run (the stale duplicate is skipped).
- two **active** runs share the key -> `MultipleMatches`, writes nothing (unchanged -- still surfaced, never silently picked). The contract makes duplicates safe by letting an operator supersede the stale one, not by auto-collapsing.
- the only-run is excluded -> `NoMatch`.

## Marking mechanism

New tenant+user-scoped endpoint `POST /api/agent-runs/{agentRunId}/exclude`, body `ExcludeRunRequest { reason, supersededByAgentRunId? }`. Validates `reason` against `RunExclusionReasons.All` (422 otherwise), 404 for a run not owned by the caller, sets the fields, returns `ExcludeRunResultDto`. The run, its `OutputJson` artifact, and its trace are preserved -- soft flag, never a delete. Scoped tighter than the automation-facing reconcile endpoint because exclusion is a deliberate operator action.

## Per-run reconciliation behavior (decision)

`POST /api/agent-runs/{id}/outcome` (keyed by run id) is the **explicit** path and is intentionally left **unfiltered** by eligibility: naming a run by id is itself the explicit allowance to act on it (decision rule 4, "reject inactive unless explicitly allowed" -- per-run-by-id is the explicit allowance). Automatic, identity-keyed selection is the path that filters. Documented here so the asymmetry is a decision, not an oversight.

## Calibration behavior

Automatic calibration candidate selection should select `ExclusionReason IS NULL` only (the seed-runbook SQL gains `AND ExclusionReason IS NULL`). Reports should distinguish: active eligible directional runs (the calibration set), null/inconclusive active runs (non-directional, excluded from accuracy but not excluded from the run set), and superseded/diagnostic runs (kept for audit, never counted). No calibration code exists yet to change; this is the rule the future read surface and runbook must follow.

## How this enables safe null-lean reruns

To rerun the three null-lean MLB games (Padres@Cardinals 823046, Twins@Rangers 822887, Pirates@Athletics 824993): generate the new run (future budgeted slice), then `POST /{old-run-id}/exclude` with `reason=superseded, supersededByAgentRunId=<new run>`. The old run is preserved for audit; the matcher then sees a single active run for that game and reconciles cleanly. No deletion, no ambiguity. (Not executed this slice -- no rerun occurred, so the three runs remain active.)

## Deferred (explicitly)

- batching/caching of analysis across users; cache/batch reuse.
- game-weighted vs run-weighted calibration math.
- buyer-visible run history / lifecycle UI / dashboard.
- retention policy and any hard-delete path.
- automatic supersession at generation time (today supersession is an explicit operator call, not auto-set when a rerun is created).
- WNBA support; scheduled settlement automation; model/prompt changes.
- a FK or self-reference constraint on `SupersededByAgentRunId` (kept as loose provenance).

## Migration / compatibility notes

- `20260615191124_AddRunMatchEligibility`: two nullable columns, no backfill, clean Down. Applied to local dev (129/129 rows active). Existing reconciled runs and the four prior outcomes/evaluations are untouched (outcomes/evals stayed 12/12).
- New generated runs default to active (`ExclusionReason` unset). No change to run creation, the analyzer, the pure matcher, the evaluator, the lean/LeanSide contract, confidence, or buyer surfaces.

## Recommended next slice

Unchanged primary action: **reconcile the 7 directionally usable MLB runs after settlement** (still pending tonight's games) via the proven statsapi path. If/when the three null-lean games are rerun, use the new `/exclude` (reason=superseded) on the old runs first, then reconcile the replacements. WNBA remains deferred; automatic-supersession-at-generation and calibration-selection-by-eligibility are follow-ups, not yet needed.
