# Outcome Reconciliation Contract v1

**date:** 2026-06-09
**status:** contract / spec only. docs only. no runtime code, no migration, no schema change, no artifact-contract/buyer-projection/confidence/posture/lean/prompt/model/cost change, no outcome-provider choice, no automated settlement.
**scope:** define the contract for matching generated decision artifacts to real game outcomes later, so a future runtime slice can implement settlement reliably, auditably, and sport-aware.

## Purpose

Decision Artifact Persistence Review v1 found the reconciliation storage is already scaffolded (AgentRunOutcomes, AgentRunEvaluations, a manual `/outcome` endpoint, the deterministic `RunEvaluator`) but two things block reliable automated reconciliation: there is no stable external game/event id on a run, and matching by display names is fragile. This slice specifies the match key, outcome record, evaluation rules, and calibration feedback needs. It implements none of them.

Primary question: what must an artifact, persisted run, and outcome record contain so DAI can later reconcile leans against completed games reliably, auditably, and sport-aware?

## Scope

- read-only inspection of the existing scaffold (entities, endpoint, evaluator, status constants).
- define: stable game/event identity, outcome record contract, evaluation contract, calibration feedback-loop contract, manual-vs-automated stages, per-sport implications, storage posture, deferred decisions.

## Non-goals

- no runtime code, EF migration, schema, or artifact-contract change.
- no outcome-provider choice or integration; no automated settlement; no scheduled job.
- no buyer-projection/confidence/posture/lean/prompt/model/cost/source/probe-refresh change.
- no Postgres migration; no pricing/Stripe/auth/tenant/dashboard/deployment change. no Jera change. no push.

## Existing reconciliation scaffold

Built today (manual), grounded in code read this slice:

- entities: `AgentRunOutcome` (AgentRunKey, OutcomeStatus, HomeScore?, AwayScore?, Source, SourceRef?, ResolvedUtc, Notes?; unique per run; never cascade-deleted) and `AgentRunEvaluation` (AgentRunKey, EvalStatus, LeanSide, WinningSide, EvaluatedUtc; unique per run).
- endpoint: `POST /api/agent-runs/{agentRunId}/outcome` accepts `RecordOutcomeRequest { OutcomeStatus, HomeScore?, AwayScore?, Source, SourceRef?, ResolvedUtc?, Notes? }`. It validates `OutcomeStatus` against `OutcomeStatuses.All`, is scoped to tenant only (the code comment notes "future automation won't have a userkey"), enforces one outcome per run (409), and writes the outcome plus a derived evaluation in one transaction.
- status taxonomy today: `OutcomeStatuses` = { home_win, away_win, draw, cancelled, postponed }; `EvalStatuses` = { correct, incorrect, inconclusive }. "unresolved" is expressed by the absence of an evaluation row, not a stored value.
- evaluator: `RunEvaluator.WinningSide` maps home_win->"home", away_win->"away", else null. `RunEvaluator.Evaluate(leanSide, outcomeStatus)`: if leanSide is null OR winning is null -> inconclusive; else leanSide == winning -> correct, else incorrect. Pure, deterministic, no i/o.
- usable fields already present: AgentRunId (GUID), promoted Competition, GameDate, LeanSide, Status, timestamps; the full artifact (raw/analyzer confidence, evidenceRichness, posture, lean, signalAvailability, signalFollowUps, pipelineSteps) inside OutputJson.

Fields missing for automated reconciliation: a stable external game/event id on the run; a scheduled UTC start time (GameDate is date-only); normalized team identifiers (the run stores display names; the seeded Teams table is not FK'd to runs); a provider/source name captured at generation time; expanded settlement statuses (final/suspended/void/unknown); separate settledAtUtc/sourceFetchedAtUtc audit timestamps.

Assumptions the current evaluator makes: leans are moneyline-style directional (home/away); a single outcome per run; LeanSide is the denormalized truth (no OutputJson parse needed); posture does not affect whether a lean is evaluated.

## Artifact-to-game matching requirements

The minimum reliable game-identity contract for automation is a stable, provider-scoped event identity captured at generation time, not display names:

- require an external game/event id (`externalGameId`) plus the `sourceProvider` that issued it (an id is only unique within its provider). Capture it on the AgentRun at generation time, ideally from the same provider that grounded the signals, so the artifact and a later settlement reference the same event.
- require a normalized home/away team reference (stable team key or provider team id), not just a display name.
- require `scheduledStartUtc` (UTC start: tip-off / first pitch / kickoff) to disambiguate doubleheaders and split-squad / same-day rematches that share teams and date.
- require `competition`/league and `season` (already have competition; season is needed because game ids and team ids recur across seasons).
- acceptable fallback for manual/dev only: `(competition, scheduledStartUtc-or-gameDate, normalized home, normalized away)`. Explicitly not acceptable for automated settlement.

Why display names are insufficient: names drift and collide. The repo already carries a `RenameOaklandToSacramento` migration -- proof that even pro team names change. College and women's sports compound it ("Miami (FL)" vs "Miami (OH)", multiple "St." schools, abbreviations), providers name teams differently, relocations/rebrands happen, and doubleheaders break (competition, date, names) uniqueness. An external event id resolves all of these; names cannot.

## Stable game/event key contract

- canonical key (automation): `(sourceProvider, externalGameId)` -- a provider-scoped, immutable event identity stored on the run at generation time and on the outcome at settlement time; reconciliation joins on equality.
- supporting identity: `scheduledStartUtc`, `season`, normalized `homeTeamRef`/`awayTeamRef`.
- fallback key (manual/dev): `(competition, gameDate, normalized home, normalized away)` with a recorded `matchConfidence` and a human in the loop.
- rule: automation must never auto-evaluate a run it cannot match on the canonical key; an unmatched run is marked needs-review, never guessed.

## Outcome record contract

An outcome record must be able to carry (contract requirements; not implemented here -- several are additions to today's `AgentRunOutcome`):

- linkage: `agentRunId` (today: via AgentRunKey), `externalGameId`, `sourceProvider` (today: free-text `Source`/`SourceRef`).
- game identity echo: `competition`/league, `season`, `scheduledStartUtc`, `homeTeamRef`/`homeTeamName`, `awayTeamRef`/`awayTeamName`.
- result: `homeScore`, `awayScore` (today: present), `winningSide` (derived), `status` in { final, postponed, cancelled, suspended, void, unknown } plus the existing directional finals home_win/away_win/draw -- the taxonomy should separate the settlement state (final/postponed/...) from the directional result (home/away/draw) rather than overloading one field.
- audit/time: `settledAtUtc` (when the real game settled), `sourceFetchedAtUtc` (when DAI read it), `resolvedUtc` (today), `recordedByUserKey` or `automated` flag, `correlationId`.
- quality: optional `sourceQuality`/`sourceConfidence` (e.g. official vs unofficial feed), `matchConfidence` for fallback matches.
- manual: `notes`/`manualOverrideReason` (today: `Notes`).

Today's `AgentRunOutcome` already covers status (subset), scores, source, sourceRef, resolvedUtc, notes. The contract adds the stable id, scheduled start, normalized team refs, the settlement-vs-direction split, and the extra audit timestamps.

## Evaluation contract

Keep the existing deterministic rule and clarify the edges (contract, not code):

- correct: a directional lean exists (home/away) AND the outcome is a settled directional final AND lean == winning side.
- incorrect: directional lean exists AND settled directional final AND lean != winning side.
- inconclusive: no lean (lean null); OR no directional winner (draw/tie); OR not settled (postponed/cancelled/suspended/void/unknown); OR teams/game could not be matched confidently.
- posture monitor/wait with a lean present: still evaluate the lean. Posture is a presentation/aggression axis, not whether a directional read exists; a "slight lean" is a directional read. Posture becomes a calibration slice, not a correctness input.
- no lean: inconclusive (already).
- unmatched teams/game: do not evaluate; mark needs-review. Automation must not fabricate a match to force a verdict.
- postponed/cancelled/suspended: no correctness yet; if the game is later rescheduled and resettled, the run may be re-matched (the one-outcome-per-run rule must allow correcting an unsettled placeholder, not double-counting).
- buyer-advertised strength must NOT affect correctness. Correctness is lean direction vs outcome only. Advertised strength, raw confidence, and evidence sufficiency are calibration slicing dimensions, never inputs to correct/incorrect. This preserves the two-axis doctrine (Confidence Calibration Rules v1): direction is judged; strength is calibrated.
- calibration counting: accuracy denominators count only correct/incorrect; inconclusive/void/unmatched are retained for audit but excluded from accuracy rates.

## Calibration feedback-loop contract

Reconciliation must preserve (and make groupable) these dimensions so the delayed feedback loop can recalibrate later:

- raw confidence and analyzer confidence (distinct -- the learning loop compares which was closer), evidenceRichness, evidence-sufficiency tier, advertised strength, advertisedStrengthReason/humility reason, posture, lean side, competition/season, a signal-availability summary, source-degradation flags, model cost (when persisted -- entry 22), generatedAtUtc, settledAtUtc, outcome status, evaluation result.
- these already live in the immutable, persisted artifact (OutputJson) plus the run/outcome/evaluation rows; evidence-sufficiency tier and advertised strength are currently derived in the Angular projection (entry 23) but are re-derivable server-side from persisted evidenceRichness + confidence. The contract requires they remain re-derivable or be promoted; it does not require promotion now.

Explicit framing:

- outcome feedback arrives later than artifact generation (hours to days). Both generatedAtUtc and settledAtUtc are required so latency and as-of grouping are possible.
- calibration should group results by evidence-sufficiency tier, sport, posture, raw confidence band, and advertised strength.
- raw confidence must remain available because buyer presentation is gated (a raw High shown as advertised Medium). Calibrate on raw confidence; present on advertised strength. This is exactly why Evidence-Sufficiency Band Gate v1 preserved raw confidence -- this contract is its downstream consumer.

## Manual vs automated reconciliation stages

- Stage 0 (now, already possible): manually attach an outcome to a run via the endpoint; the deterministic evaluator runs and persists an evaluation; inspect the evaluation; use settled runs for calibration analysis. No new work needed to start collecting reconciled data manually.
- Stage 1 (contract-first, later): add the stable match-key fields (migrations) and a matcher that joins runs to settled games on the canonical key; still no provider chosen; settlement entered manually or via a thin import.
- Stage 2 (later): integrate a settlement provider (provider-agnostic contract; choice deferred).
- Stage 3 (later): scheduled settlement job that fetches finals and records outcomes; idempotent, audited, never guessing matches.

Before any automation, Stage 0 already supports manual reconciliation and calibration; the recommendation is to begin manual reconciliation to learn outcome-source reliability before automating.

## Per-sport implications

- sport-agnostic fields: agentRunId, sourceProvider, externalGameId, competition, season, scheduledStartUtc, home/away team refs, status, scores, winningSide, leanSide, evalStatus, the calibration dimensions.
- sport-specific later: winner-determination rules (overtime, shootouts, mercy/run rules, suspended-game resumption), draw possibility (soccer; rare NFL ties -- not NBA/MLB), and a different evaluation function if leans ever become spread or total reads (winning side becomes cover / over-under, not home/away). Current leans are moneyline-style; the evaluator is correct for that and should be versioned if lean types expand.
- current NBA/MLB: moneyline direction, no draws in practice; the existing evaluator fits.
- future NFL/CFB, NCAAB (men/women), WNBA: same contract; NHL deferred unless demand. Stable event identity matters most for college and women's sports because of naming ambiguity, more same-name programs, and thinner/divergent provider coverage -- display-name matching breaks hardest there, so the external-id requirement is non-negotiable for those before enabling them.

## Storage posture

Stay on SQL Server (Decision Artifact Persistence Review v1, ledger entry 24). This contract is database-agnostic. Changes that WILL require migrations when the runtime slice lands (do not implement now): add `externalGameId`, `sourceProvider`, `scheduledStartUtc`, and team-ref columns to AgentRun (and echo on AgentRunOutcome); split/extend the outcome status taxonomy and its check constraint; optionally promote calibration dimensions (raw confidence band, evidence tier, advertised strength) to indexed columns for grouping. None of these require Postgres; all fit SQL Server. Postgres remains deferred and evidence-gated.

## Deferred decisions

- stable game/event match key + outcome record extensions: contract defined here; implementation (migrations + matcher) deferred to Outcome Reconciliation Runtime v1. Recorded as new ledger entry 25.
- outcome settlement provider + automated/scheduled settlement: deferred; provider-agnostic contract only.
- promotion of calibration dimensions to columns vs derive-on-read: deferred; re-derivable today (entries 22, 23).
- calibration-driven threshold changes: still gated on reconciled-outcome evidence (entry 12) -- this contract is the path to producing that evidence, but changes nothing about it.

## Recommended next slice

Per-Sport Buyer Validation Matrix v1 (planned step 6) can proceed in parallel using existing persisted runs grouped by competition. The implementation that this contract enables is Outcome Reconciliation Runtime v1 (step 7): add the stable-id columns + matcher, expand the status taxonomy, and (optionally) a thin manual/import settlement path -- still provider-agnostic. Recommend also beginning manual Stage 0 reconciliation now to learn outcome-source reliability before automating.

## What was not changed

- no runtime code, EF migration, schema, or artifact contract
- no outcome-provider choice/integration, no automated settlement
- no buyer projection, confidence, posture, lean, or decision logic
- no prompt, model-call, or cost-guardrail change
- no source/probe-refresh/Postgres/pricing/Stripe/auth/tenant/dashboard/deployment change
- no Jera change

## Verification / unknowns

- all scaffold claims are grounded in `AppDbContext.cs`, the AgentRun/AgentRunOutcome/AgentRunEvaluation entities, `AgentRunsController.RecordOutcome`, `RunEvaluator`, and the `OutcomeStatuses`/`EvalStatuses` constants (read this slice).
- not determined (runtime state, not code): whether any outcome/evaluation rows exist yet in the dev database. Expectation: few or none (no automated source). Verify later with a read-only count if needed; not required for this contract.
- the exact future column set is a recommendation; the runtime slice will finalize names/types when it writes the migration.
