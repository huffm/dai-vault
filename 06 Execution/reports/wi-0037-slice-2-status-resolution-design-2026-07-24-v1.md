---
title: "WI-0037 Slice 2 Status Resolution Design 2026-07-24 v1"
type: "evidence-report"
date: "2026-07-24"
status: "design complete -- decomposition into two implementation slices recommended; NO implementation authorized"
project: "DAI"
slice: "WI-0037 Slice 2 planning: canonical date-bracketed status resolution design"
repos:
  dai: "unchanged (read-only surface mapping at main 0a9129b)"
  dai-vault: "docs-only; local branch wi/0037-game-state-correctness-slice-2-planning from 14c3926"
tags:
  - system-development
  - sports-v1
  - correctness
  - architecture
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/daily-evidence-late-slate-reevaluation-2026-07-23-v1.md"
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
---

# wi-0037 slice 2 status resolution design 2026-07-24 v1

## 1. executive conclusion

The production C# identity core is strong: `MlbEventResolver` is fail-closed and
doubleheader-safe (24 pinned scenarios), both its callers are provably date-scoped, and
the market-contrast state gates are Slice-1-corrected and fully pinned. The real Slice 2
work concentrates in three places: (a) `check-settlement-finals.ps1` fetches by bare
`gamePks=` and flattens all `dates[]` buckets, so it FALSE-DEFECTS the legitimate
postponed-original + makeup two-bucket class -- the exact July 23 identity shape -- and
no test pins that case; (b) there is NO canonical date-bracketed operator status script;
the corrected discipline ("`date=` + gamePk, never `dates[0]`") exists only as prose in
two report bodies; (c) `OddsScheduleClient` collapses legitimate same-date doubleheaders
via `DistinctBy(Date)` on a reference surface with zero test coverage. "Exactly one
match" must be a STAGED policy, not naive uniqueness. Recommended: decompose into two
implementation slices, scripts/operator first.

## 2. surface inventory and call graph

Authoritative StatsAPI identity/status chain (C#): `MlbStarterClient` fetches
`/api/v1/schedule?sportId=1&date={gameDate}&hydrate=probablePitcher`
(`MlbStarterClient.cs:244` single, `:306` batch), flattens buckets (`:278`, `:338`),
resolves per candidate through `MlbEventResolver.Resolve` (`:288`, `:344`), grounds via
`GroundFromResolutionAsync` (`:359-450`, ambiguity refusal `:378-387`), and stores the
resolved game's `DetailedState`/start (`:348-349`). Consumers:
`MarketContrastSourceAdapter` (batch, `:173-175`; state gates `:348-357`; facts
`:602/:627`), `SportsRetriever.RetrieveAsync` (delegates with `RequestedGamePk`,
`SportsRetriever.cs:107-115`), tool `pitching.mlb.probable_starters`
(`RetrieveSignalHandlers.cs:57`). Bracket authority:
`OddsMarketClient.EasternDayBracket` (`:548`), consumed by events gate, adapter, and
`ProviderEventBinding` (`:445`). Reference-only odds schedule: `OddsScheduleClient`
(`:127`, `:223`) feeding `schedule.matchup_dates` and upcoming-samples. Scripts:
`check-settlement-finals.ps1` (the ONLY script resolving StatsAPI status;
`gamePks=` fetch `:246-247`, flatten `:255`, duplicate-DEFECT `:96-103`, final rule
`:111`); `preflight-settlement.ps1` reads local API rows only (no StatsAPI). Python
agent-service: no HTTP client exists; the planner validates a supplied
`schedule_status` against a fixed vocabulary (`daily_evidence_planner.py:129-132`,
`:764-768`) and never re-derives status. Fatigue proxy:
`MlbBullpenFatigueClient.cs:169/:182-187` selects the last Final game for workload
context only.

## 3. invariant analysis

Refined parent invariant (strengthened, not weakened): every authoritative MLB
game-status decision must resolve through a STAGED policy --

1. fetch may legitimately return multiple raw records for one gamePk across `dates[]`
   buckets (postponed original + makeup);
2. a deterministic frozen Eastern date bracket filters buckets FIRST;
3. exact `gamePk` resolution applies WITHIN the bracket;
4. exactly one authoritative candidate must remain; same-pk entries OUTSIDE the bracket
   are reschedule context (`rescheduledFrom`/`rescheduleDate`), never duplicates;
   two entries for one pk INSIDE one bracket are a true identity DEFECT.

Prohibitions confirmed: no positional bucket selection (`dates[0]` -- zero first-party
occurrences today; operator-side only), no first-match without uniqueness proof, no
date-only dedup on identity paths, no undocumented caller guarantees, no silent
substitution. Naive "exactly one match anywhere in the response" is WRONG for this
provider -- it is precisely what makes the finals guard false-DEFECT the makeup class.

## 4. per-surface classification

| Surface | Caller/consumer | Date scope | Identity rule | Classification | Disposition |
|---|---|---|---|---|---|
| `MlbEventResolver.cs:64/:86` | MlbStarterClient x2 | none in callee; both callers fetch `date=` | exact pk, fail-closed, ambiguity refusal | `IMPLICIT_CALLER_INVARIANT` | enforce/document the date-scoped-input contract at the callee seam (Slice 2-ii) |
| `MlbStarterClient.cs:244/:306` fetch+flatten | adapter, retriever, tool | explicit `date=` | resolver above; `byGamePk` last-write-wins on duplicate input pk (`:341-350`) unreachable via adapter validation ("duplicate gamePk in slate") | `COMPLIANT_TEST_GAP` | batch-boundary tests incl. duplicate-input behavior (2-ii) |
| `MarketContrastSourceAdapter` state gates `:348-357` | screen pipeline | via TargetDate | Slice-1 corrected, pinned matrix | `COMPLIANT_NO_CHANGE` | none |
| `EasternDayBracket` (`OddsMarketClient.cs:548`) | gate/adapter/binding | DST-correct ET | single authority, pinned | `COMPLIANT_NO_CHANGE` | none |
| `OddsScheduleClient.cs:223` pair path | `schedule.matchup_dates` tool | ET-date derived | `DistinctBy(Date)` collapses legit same-date DH | `CONFIRMED_DEFECT` (D2; reference surface) | DH-safe dedup key + first tests (2-ii) |
| `OddsScheduleClient.cs:127` sampler | dev upcoming tool | ET-date derived | date:home:away key collapses DH | `CONFIRMED_DEFECT` (D2 minor) | same slice, same fix pattern (2-ii) |
| `MlbBullpenFatigueClient.cs:182-187` | fatigue signal | startDate/endDate | last-Final selection, proxy only | `OUT_OF_SCOPE` (test-gap noted) | none in WI-0037 |
| `SportsRetriever` / `ProbeRefreshExecutor` | orchestration | delegates | no own status decision | `OUT_OF_SCOPE` | none |
| `check-settlement-finals.ps1:246/:255/:96-103` | settlement gate | NONE (`gamePks=`, flatten) | false-DEFECTs legit two-bucket makeup; multi-bucket untested | `CONFIRMED_DEFECT` (D1) | bracket-staged resolution + fixtures (2-i) |
| `preflight-settlement.ps1` | write-safety gate | n/a | no status resolution (local rows only) | `OUT_OF_SCOPE` | none |
| python planner `schedule_status` | board eligibility | n/a | typed consumer, fixed vocabulary | `OUT_OF_SCOPE` (aligned) | none |
| operator ad-hoc status checks | pre-capture/recheck | ad hoc | July 23: bare pk + `dates[0]` | `OPERATOR_GAP` (G1) | canonical script + pattern doc (2-i) |
| pattern doctrine | settlement-readiness-discipline-v1 | n/a | duplicate-DEFECT codified; date-bucketing ABSENT | `OPERATOR_GAP` (G1) | codify bracket discipline (2-i) |

## 5. confirmed defects, implicit assumptions, operator gaps

- **D1 (medium, fail-closed direction): finals-guard two-bucket false-DEFECT.**
  `check-settlement-finals.ps1` queries `gamePks=` with no date filter and flattens all
  buckets; a postponed-original + makeup pk yields `found.Count = 2` -> DEFECT exit 3,
  blocking a legitimate settlement. The duplicate test pins only the single-bucket
  hazard case; the multi-bucket legit case is unpinned. Would have false-DEFECTed
  823042's class had its settlement been attempted through a date-expanded payload.
- **D2 (low-medium, reference surface): OddsScheduleClient DH collapse.** Pair path
  `DistinctBy(e => e.Date)` (`:223`) drops one game of a legitimate same-date
  doubleheader after pair filtering; `:127` similar for the dev sampler. Zero tests
  exist for this client. Not on the authoritative identity path (provider-event binding
  uses the events gate + EasternDayBracket instead), but `schedule.matchup_dates`
  cannot surface both DH games.
- **D3 (low, rider): null/absent frozen `ScheduleState`** NREs in
  `NormalizeScheduleState` (slate validation `:306-314` never checks the field).
  Crash-not-accept; pre-existing (Slice 1 review Finding 2).
- **G2 (implicit assumption): resolver date scope.** `MlbEventResolver` cannot enforce
  that its input games came from a bracketed fetch; safety rests on both callers'
  `date=` URLs. Unenforced invariant, currently true.
- **G1 (operator gap, the realized July 23 cost):** no canonical date-bracketed status
  script exists (`scripts/**/*status*.ps1` -> none); the corrected rule lives only in
  two report bodies; the misread used a bare-gamePk query and read `dates[0]` (the
  June-25 bucket) -- reconstructed verbatim from the late-slate reevaluation record.
- **No DESIGN_CONTRADICTION found**: all consumers want the same staged semantics.
- First-party positional `dates[0]`: NONE (re-verified this session).

## 6. cross-runtime architecture (decision)

- **Option A (one canonical runtime resolver for all paths): rejected.** PowerShell
  operator/settlement scripts are deliberately dependency-free and offline-testable
  (`-ScheduleJsonPath` seam); forcing them through a .NET process adds startup cost,
  deployment coupling, and new failure modes for zero semantic gain.
- **Option C (C# resolver + thin script adapter/CLI): rejected for now.** Same coupling
  concerns; a CLI host is real new surface; operator checks must work when the platform
  is down (settlement gates ran today with the API stopped).
- **Option B (shared versioned contract + fixture vectors): SELECTED.** One versioned
  resolution contract -- proposed `game-status-resolution/1.0` -- with one reason
  vocabulary and ONE canonical JSON fixture corpus consumed by BOTH the xunit suite and
  the PowerShell test harness, mirroring the workspace's proven cross-language vector
  discipline (WI-0035/0036). MLB schedule semantics stay niche-owned; scripts and C#
  implement the same staged policy independently but conformance is machine-checked
  against identical vectors.

**Contract sketch (`game-status-resolution/1.0`):** inputs = raw schedule payload +
frozen Eastern bracket date + exact gamePk; staged resolution per section 3; typed
outcomes = `resolved` (one record: normalized status, start, bucket date, reschedule
context) | `refused` with reason from a closed vocabulary:
`bracket_missing`, `game_not_in_bracket`, `duplicate_in_bracket`,
`bucket_malformed`, `status_malformed`, `identity_mismatch`. Reschedule context is
DATA (`rescheduled_from`/`rescheduled_to`), never a refusal.

## 7. required scenario corpus (fixture vectors)

24 deterministic fixtures, real-shaped (the two-bucket vectors derive from the stored
July 23 823042 evidence; no live calls): (1) normal single game -> resolved/scheduled;
(2) same-date different-team games -> resolved via pk; (3) legitimate DH, pk game 1 ->
resolved; (4) same teams same date distinct pks -> each resolves, no collapse;
(5) postponed original + scheduled makeup (two buckets, one pk) -> bracket selects
makeup, resolved/scheduled, reschedule context attached; (6) postponed original +
COMPLETED makeup -> bracket selects makeup, resolved/final; (7) historical + current
buckets sharing pk, bracket = historical date -> resolves the original with
rescheduled_to context; (8) one pk under an unexpected response shape (extra empty
buckets) -> resolved; (9) true duplicate pk within ONE bucket -> refused
duplicate_in_bracket; (10) missing pk -> refused game_not_in_bracket; (11) undated
invocation -> refused bracket_missing; (12) Eastern boundary 23:59 ET -> bracket keeps
the ET date; (13) UTC date differs from ET game date (10pm ET) -> ET bracket wins;
(14) rescheduled start same bucket -> resolved with new start; (15) suspended ->
resolved/suspended (consumer decides screenability); (16) cancelled ->
resolved/cancelled; (17) in progress -> resolved/in_progress; (18) official final
(abstract Final + coded F) -> resolved/final; (19) malformed/absent date bucket ->
refused bucket_malformed; (20) malformed/absent status -> refused status_malformed;
(21) null/absent frozen ScheduleState (slate-side) -> typed validation rejection, see
section 8; (22) alias `Warmup` -> normalized pregame; (23) finals guard over legit
makeup/final pair -> READY for the bracket game, no DEFECT; (24) finals guard over a
true in-bracket duplicate -> DEFECT retained. Each fixture specifies raw payload,
bracket date, target pk, expected record, expected normalized status, expected
outcome/reason, and consuming surfaces (guard, resolver, operator script).

## 8. nullable `ScheduleState` disposition (rider)

**Decision: typed INVALID INPUT at slate validation, inside Slice 2-ii.** Null, absent,
or whitespace frozen `ScheduleState` must be rejected by the existing request
validation ("candidate schedule state is required") BEFORE normalization -- not mapped
to `unknown`. Rationale: accepting null as `unknown` would conceal malformed input
behind a misleading `caller_state_mismatch` and turn a caller bug into a per-candidate
elimination; validation rejection is fail-closed, precise, and matches the closed-input
contract. Unrecognized NON-EMPTY strings keep the existing pinned behavior (normalize
to `unknown` -> fail-closed mismatch). It is a hardening prerequisite for trusting the
frozen slate, small enough to ride 2-ii; not a separate slice.

## 9. slice decomposition (decision)

**Recommendation: TWO implementation slices** (not one, not the 2A/2B/2C three-way):

| Proposed slice | Scope | Dependencies | Expected paths | Test proof | Risk |
|---|---|---|---|---|---|
| **2-i: contract, fixtures, finals guard, operator script** | `game-status-resolution/1.0` contract doc + canonical JSON fixture corpus; finals-guard staged-bracket correction (fetch by `date=` + pk or bracket-filter before uniqueness; keep in-bracket duplicate-DEFECT); NEW canonical operator status script (`check-game-status.ps1`, date-bracketed, typed output); pattern-doc codification of the bracket discipline | none | `scripts/dev/sports/check-settlement-finals.ps1`, new `check-game-status.ps1`, new fixtures dir, `test-check-settlement-finals.ps1`, new script tests; vault pattern + contract docs | RED: fixture 23 false-DEFECTs today; fixture 24 stays DEFECT; operator-script vectors | low-medium: settlement-gate behavior change, mitigated by keeping every existing pinned case green |
| **2-ii: C# conformance + hardening** | resolver date-scoped-input seam made explicit (enforced or contract-documented + tested at the callee); `OddsScheduleClient` DH-safe dedup keys + first unit tests; batch-boundary tests (duplicate input pk); null-`ScheduleState` validation rider; xunit conformance runner over the SAME fixture corpus | 2-i (corpus exists) | `MlbEventResolver.cs`, `OddsScheduleClient.cs`, `MarketContrastSourceAdapter.cs` (validation only), new test files | RED: DH-collapse fixture on `:223`; null-state fixture; corpus conformance | low: reference-surface + additive tests; no screen-policy change |

Rationale: one mega-slice would span PowerShell + C# + new operator tooling + contract
authorship (poor reviewability, mixed RED domains); the three-way split leaves 2A as an
unproven floating contract with no RED. Two slices give each a real RED, keep
changed-path counts small, and put the operationally-costly surfaces (guard + operator
gap -- the class that already burned credits and would block a makeup settlement)
first. Dependency order: 2-i then 2-ii. The `r7a26` console-race hygiene candidate
stays OUTSIDE WI-0037, schedulable independently in parallel (no path overlap).

## 10. exclusions, risks, next authorization

Excluded (unchanged from the WI): caller-state changes (Slice 1 closed), broad StatsAPI
client rewrite, schedule-adapter automation (WI-0034 S3), planner-board schemas,
provider-binding changes, paid validation, schema/migrations (none needed -- all
changes are code+tests+scripts+docs). Risks: finals-guard change touches the settlement
gate (mitigate: every existing pinned case must stay green, in-bracket duplicate-DEFECT
retained, RED-first on fixture 23); DH dedup change alters `matchup_dates` output
shape for DH days (consumers are reference surfaces; document). Next authorization =
**WI-0037 Slice 2-i implementation** exactly as scoped above.
