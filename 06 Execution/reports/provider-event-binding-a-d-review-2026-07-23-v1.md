---
title: "Provider-Event Binding A-D Vertical -- Adversarial Review 2026-07-23 v1"
type: "review-report"
date: "2026-07-23"
status: "review PASSED, zero corrections required; local integration authorized by operator prompt; push NOT authorized"
project: "DAI"
slice: "WI-0035/WI-0036 provider-event binding vertical, Slices A-D adversarial review before local integration"
repos:
  dai: "reviewed range 48a2931..85af96d on local wi/0035-provider-event-binding; NOT pushed"
  dai-vault: "reviewed range 3a82af0..4cb22a2 on local wi/0035-provider-event-binding; NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - review
  - wildcard-flight
related:
  - "06 Execution/reports/provider-event-binding-slice-d-2026-07-23-v1.md"
  - "06 Execution/reports/provider-event-binding-slice-c-2026-07-22-v1.md"
---

# provider-event binding A-D vertical -- adversarial review

## independence caveat (stated first)

This review was performed in the SAME session that implemented Slice D, under an explicit
operator authorization to review adversarially from repository evidence. Every claim below
was re-established from git history, source, tests, and live command output -- not from the
implementation report -- but this is not organizationally independent review. If a future
gate demands true independence, rerun the review in a fresh session before push.

## reviewed ranges (verified live)

- dai `48a2931..85af96d`, 10 commits, linear: Slice A RED seam proof `96b935f`; Slice A
  producer `2e24782`; corrections 1A `6480a94`, 1B `54873b3`, Pass-2 `4702b51`+`4f4e726`,
  Pass-2 completion `72a0347`; Slice B `6d4cd32`; Slice C `88092eb`; Slice D `85af96d`.
- dai-vault `3a82af0..4cb22a2`, 13 commits: four pre-vertical July-22 operational records
  (`25ef999`, `2a5bc46`, `ceb6b96`, `9ef41c9` -- events-gate day + start-instant analysis,
  docs-only, $0) beneath the nine vertical records ending at the Slice D record `4cb22a2`.
  Integration of this branch necessarily carries those four records to main; reviewed and
  acceptable (they document completed $0 observation work).
- No rewrite, no partial integration, no upstream on either WI branch; both mains at their
  expected SHAs and equal to origin.

## end-to-end bound execution trace (re-established from source)

request bytes (`flightSelection.providerEventBinding`) -> `AgentRunsController.Create`
strict `ProviderEventBindingWire.Parse` on `GetRawText` + gamePk/flight-identity/date/
competition consistency (400 BEFORE run row) -> `AgentRunService` raw bytes onto
`SportsRunArtifact.ProviderEventBindingWire` (explicit json null = validly unbound) ->
`SportsRetriever` (non-mlb bound refusal; verbatim carry into `MarketSpreadInput`;
post-retrieval strict re-parse + grounded-id comparison; records
`ProviderEventBindingVerification.Verified`) -> `MarketBaseballSpreadHandler` strict
re-parse, fail-closed pre-provider -> `OddsMarketClient.GetBaseballSpreadByBindingAsync`
(internal; reachable only with a strictly-parsed binding; uncached) re-runs
`ProviderEventQualifier.Qualify` against the frozen candidate + bracket and requires
Qualified AND the identical provider event id -> typed
`ProviderEventBindingIntegrityException` (closed statuses) on every identity failure ->
controller final defense: unverified bound execution = failed row + 422, no buyer result.

Non-baseball traces: football and basketball execution pass an EXPLICIT null binding
(absence discriminator); their handlers additionally reject any supplied binding; neither
sport is forced to fabricate a binding and neither became incoherent. Ordinary unbound MLB
runs remain valid (null wire) with hardened matching.

## adversarial findings

| id | severity | finding | evidence | disposition |
|---|---|---|---|---|
| F1 | accepted residual | near-close by-id capture path (`GetBaseballSpreadByEventIdAsync`) has no frozen-binding re-verification (duplicate-id refusal only) | exhaustive caller search: only `ClosingSnapshotCaptureService`, reached only via the two near-close controller endpoints; those write market-snapshot batches and construct NO `MarketSpreadInput`, NO run, NO artifact, NO buyer result -- provably cannot bypass bound execution | documented; post-slice candidate |
| F2 | low (observability) | unbound ambiguity/orientation refusal is indistinguishable from "no market" to consumers (null grounding; distinct only in logs), and a refused match is cached null 15 min | `FetchSpreadAsync` refusal returns null exactly like no-match; caching pre-existing | documented; refine only if operators need a distinct signal |
| F3 | low (pre-existing) | near-close endpoints carry "dev-gated" comments but no `IsDevelopment` check was found on that path | grep of controller + trigger | pre-existing standing-risk family (unauthed surfaces already tracked); outside this invariant; not corrected here |
| F4 | info (by design) | bound handler ignores input HomeTeam/AwayTeam/GameDate; the binding is sole retrieval authority | controller/retriever enforce request-vs-binding consistency upstream; a direct gateway caller with inconsistent teams still retrieves only the bound event | correct authority ordering; documented |
| F5 | low (pre-existing) | `test_plan_byte_deterministic_across_runs_and_fresh_processes` is cwd-dependent (fails when pytest runs from repo root, passes from `services/agent-service`) | reproduced live during review | pre-existing harness sensitivity, not a code defect; full suite runs from the canonical cwd |
| F6 | info | vault integration carries the four pre-vertical July-22 operational records | branch topology | acceptable; docs-only $0 records |

Zero Blocker, zero High, zero Medium requiring correction. **No correction commits were
created; the reviewed tips are the implementation tips.**

## specific mandated questions answered

- **MarketSpreadInput coherence:** the binding member is POSITIONALLY REQUIRED (no default
  -- omission is a compile error, proven by reflection test) and SEMANTICALLY NULLABLE
  (explicit null = explicitly unbound). `MarketSpreadInput` is never JSON-deserialized
  anywhere in the repo and the gateway passes the typed instance, so no deserialization or
  `default()` path can bypass the requirement. Coherent; no redesign needed.
- **Bypass search:** production callers of the three baseball retrieval methods are exactly
  {capture service, unbound handler branch, bound handler branch}. All remaining
  `FirstOrDefault` uses in `OddsMarketClient` select markets/bookmakers/outcomes WITHIN an
  already-selected event, never the event. MarketContrast screen/gate/diagnostics inject the
  client for screening with the Slice A qualifier -- they do not execute runs. ProbeRefresh
  passes explicit null (unbound diagnostics). No alternate controller, job, or CLI reaches
  bound execution around the gateway.
- **Byte preservation:** the authoritative wire travels `GetRawText` -> string -> strict
  parse; nothing reserializes it on the retrieval path. The persisted request InputJson
  re-embeds the JsonElement (pre-existing Slice C audit behavior); the canonical wire is
  whitespace-free so the embedding is byte-stable, and retrieval never reads InputJson.
- **Re-verification:** re-runs `ProviderEventQualifier.Qualify` -- the single authority, no
  competing predicate -- plus an ordinal event-id equality gate; drift, substitutes,
  duplicates, reversal, bracket and whole-second rules all re-decided from live provider
  state (proven by the twelve enforcement tests).
- **Test honesty:** 1746 -> 1760 fully attributable (+8 converted gap-class tests, +3
  retriever, +3 endpoint); zero skipped; RED-first evidence reproduced in-session against
  unmodified production for the behavioral conversions; endpoint tests traverse the real
  ASP.NET host and DI path; the bound reserve cross-runtime vector passes the new strict
  controller validation unchanged.
- **Schema stability:** Slice D touched zero python files and zero schema versions;
  board 2.3 consumption and the historical board-2.2 structured rejection are unchanged
  from Slice C; WI-0034/WI-0036 shared-planner coupling untouched.

## validation evidence at the reviewed tips

- full `DevCore.Api.Tests` at dai `85af96d`: **1760/1760, 0 skipped**
- full agent-service pytest (canonical cwd): **691/691**
- flight suites focused re-run: 61/61
- `git diff --check` clean; dai working tree clean; `DevCore.Data.csproj` protected hash
  `63ef2488...` unchanged; preserved vault dirty set byte-identical to the Slice D
  preservation table (six paths, untouched)

## integration judgment

PASS. All gates of the authorization are satisfied: no Blocker/High/uncorrected-relevant-
Medium, linear history, mains unmoved, preservation intact. Local fast-forward integration
of dai then vault is authorized to proceed. **Push remains NOT authorized.**

## $0 statement

Zero paid or live provider calls during review (all tests fake-http). No db access, no
capture, no settlement. Nothing pushed; no remote branch; no PR.
