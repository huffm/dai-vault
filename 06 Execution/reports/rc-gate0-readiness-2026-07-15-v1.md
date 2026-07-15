---
title: "RC Gate 0 Readiness and Timeline Alignment v1 (2026-07-15)"
type: "evidence-report"
date: "2026-07-15"
status: "complete"
project: "DAI"
slice: "DAI RC Gate 0 Readiness and Timeline Alignment v1"
repos:
  dai: "unchanged (read-only verification + runtime rehearsal at 85a8831)"
  dai-vault: "docs-only"
tags:
  - release
  - readiness
  - operations
related:
  - "06 Execution/plans/v1-rc-drill-package-v1.md"
  - "06 Execution/plans/v1-concierge-operations-runbook-v1.md"
  - "06 Execution/plans/v1-to-v2-release-sequence-v1.md"
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
---

# rc gate 0 readiness and timeline alignment v1

Documentation and read-only operational-readiness slice, executed 2026-07-15. NOT an
implementation slice, NOT the RC drill. Zero code changes, zero paid calls, zero sports
runs, zero DB writes, zero reconciliation, zero Stripe transactions, zero outreach.

## 1. verdict

**READY WITH OPERATOR ACTION.**

Technical readiness for RC Gate 1 on Friday 2026-07-17 passes on every criterion this
slice can verify: repository state matches the authorized release state; every runbook
command, path, endpoint, parameter, and port verifies against dai `85a8831`; all
required configuration is present; startup and shutdown rehearsed successfully from the
runbook and returned to cold; the drill workspace is prepared; no release-critical
defect was found.

Outstanding operator-owned input: **no evidence of an approved Stripe TEST-MODE payment
link exists** in the vault, the workspace, or operator notes (status:
operator-action-required). This does not block the technical portions of Friday's
drill, but if still outstanding at drill time the eventual Gate 1 verdict is CAPPED at
CONDITIONAL PASS on the entitlement criterion (freeze-doc criterion 1).

Also operator-provided at drill time (normal procedure, not a defect): the drill-day
authorization document naming candidates and caps (runbook 1.7). No standalone written
"RC Gate 1 prompt" artifact exists in either repo or the prompt ledger; the drill
package + the drill-day authorization ARE the authoritative inputs. The exact Gate 1
invocation is in section 10.

## 2. repository state (verified 2026-07-15, before and after: identical)

| repo | main | origin/main | notes |
|---|---|---|---|
| dai | `85a8831` | `85a8831` | retained branches at documented hashes: wi/0011 `140b5a2`, wi/0012 `7152818`, wi/0013 `85a8831`; working tree carries ONLY the documented DevCore.Data.csproj line-ending phantom |
| dai-vault | `02a5d30` (before commit of this record) | `02a5d30` | exactly the two documented intentionally untracked files present, untouched |

## 3. timeline alignment (phase 1 corrections)

Operator decision recorded: outreach deferred; prospect/buyer contact, free sample
delivery, and real buyer delivery NOT authorized; no commercial action inferred from
technical readiness; commercial activation requires a separate explicit operator
decision. August 7 converted from committed milestone to conditional target ("earliest
planning target ... if commercial activation is separately authorized"). RC outcome
recorded as proving operational readiness only. Committed technical sequence: Gate 0
readiness -> Gate 1 pregame drill -> read-only settlement preflight -> separately
authorized reconciliation -> recap verification -> final RC verdict -> explicit
operator decision. Cloud/multisport remain forecast/option-horizon; WI-0014..0019
remain proposed, NOT minted, NOT promoted.

Documents corrected (docs-only):

1. `06 Execution/plans/v1-to-v2-release-sequence-v1.md` -- new governing section 0
   (commercial posture); section 3 committed horizon rewritten technical-only with the
   deferred commercial track; section 8 commercial actions marked DEFERRED; V1.0
   commercial epic marked deferred.
2. `06 Execution/reports/release-architecture-review-2026-07-14-v1.md` -- operator-
   decision supersession block covering sections 1, 7, 10 (report text preserved as
   the 07-14 record).
3. `06 Execution/plans/v1-release-critical-path-2026-07-14-v1.md` -- milestone table:
   first paid pilot -> conditional target; dated-path outreach/pilot rows -> deferred/
   conditional.
4. `04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md` -- RC criterion
   1 note: real paid entitlement required for pilot validation, 08-07 conditional.
5. `04 Products/sports-v1/buyer-validation-brief-v1.md` -- status: outreach deferred,
   activation requires separate explicit operator decision (plan retained).

The long-term commercial gates (payment as the only validation, Stripe = truth) are
unchanged; payment is NOT redefined as unnecessary.

## 4. drill-package verification (phase 2)

Every command, path, endpoint, environment variable name, script parameter, and output
location in the runbook, drill package, and ledger template verified against dai
`85a8831`:

VERIFIED: `scripts/stop-platform-api.ps1` (exit codes 0/2/3 as documented);
`scripts/dev/sports/check-settlement-finals.ps1`;
`scripts/dev/sports/preflight-settlement.ps1` (`-Competition -GamePks
-RequireUnreconciled -FailOnWarnings` all real parameters);
`platform/dotnet/DevCore.Api` (+ `appsettings.Development.json`, port 5007 via
launchSettings); `services/agent-service` (`.env`, `.venv`, `main.py`, `/api/ping`
route, uvicorn invocation); `apps/sports-app` (`npm start` = `ng serve`, default 4200;
4201 covered by the `stop-all-dev.ps1` port convention, which also documents gRPC
50051); API routes `POST /api/agent-runs` (WI-0013 duplicate 409 guard present, body
fields RunType/Input.GamePk verified), `GET source-readiness`, `recent`,
`prompt-route-calibration/rows`, `{id}/prompt-trace`, `{id}/brief[/markdown]`,
`{id}/recap[/markdown]`, `{id}/evaluation`, `POST reconcile`, `GET /health`,
`GET /api/competitions/{code}/matchup-dates` (SportsReferenceController);
`DAI_MLB_REGISTRY_PROMPT_CANARY` (real, correctly ABSENT from `.env`);
`model_metering.PRICING` covers gpt-4o-mini + gpt-4.1-mini with named unpriced states.

OPERATOR-DEPENDENT: drill-day authorization document (candidates + caps, runbook 1.7);
approved Stripe test-mode payment link (section 1); buyer email template (concierge
manual step); `.env`/appsettings secret VALUES (presence verified, values never read).

STALE: none found. MISSING: none found. UNCLEAR: none.

Documentation-only defects requiring correction inside the drill docs: NONE (the phase
1 posture corrections were the only document changes needed). No code or script defect
discovered.

## 5. drill workspace (phase 3 -- local, uncommitted, outside both repos)

Root: `C:\Users\trolo\source\repos\dai-workspace\rc-drill-2026-07-17\`

- `rc-record-working-v1.md` (RC record working file, criteria table + caps + verdict)
- `delivery-ledger-2026-07.csv` (template header, verbatim)
- `operator-time-log-2026-07.csv` (template header, verbatim)
- `stripe-receipt-alias-map-PRIVATE.md` (placeholder; aliases only; no secrets present)
- `simulated-delivery\` (brief/recap markdown destination; README with naming rule)
- `command-checklist.md` (full Friday command sequence, repo-verified)
- `stop-conditions-checklist.md`
- `final-report-skeleton.md` (maps to the committed RC record format)
- `gate2-preflight-skeleton.md` (read-only preflight vs separately authorized writes)

No customer names, emails, receipts, or secret values in any artifact.

## 6. configuration presence (phase 4 -- presence only, values never read)

| area | key(s) | status |
|---|---|---|
| database | ConnectionStrings:Sql (appsettings.Development.json) | present (non-empty) |
| DevCore API | appsettings.Development.json + launchSettings 5007 | present |
| agent service | services/agent-service/.env + .venv + main.py | present |
| sports application | src/environments/environment[.development].ts apiBaseUrl | present |
| registry v2 route | canary mechanism in code; flag ABSENT from .env (required) | present/correct |
| model configuration | metering PRICING covers configured models (4o-mini, 4.1-mini) | present |
| OpenAI access | OPENAI_API_KEY in .env | present (non-empty) |
| odds provider | OddsApi:ApiKey in appsettings.Development.json | present (non-empty) |
| auth / dev access | AzureAd:TenantId+Audience; Dev:EnableBypassAuth(+keys) | present |

No missing required secret. Friday is not blocked on configuration.

## 7. stripe dependency (phase 5)

**operator-action-required** -- no approved test-mode payment link identified in the
vault, workspace root, or `.local` operator notes; Stripe itself was NOT accessed. The
technical drill proceeds regardless; the Gate 1 entitlement criterion caps at
CONDITIONAL PASS until the operator supplies the approved test-mode link.

## 8. doubleheader status (phase 6 -- free statsapi only)

Verified from `statsapi.mlb.com/api/v1/schedule?sportId=1&date=2026-07-17`:
Tampa Bay Rays @ Boston Red Sox, Fenway Park, Friday 2026-07-17; game 1 gamePk
**824766** (17:35Z, gameNumber 1), game 2 gamePk **824737** (23:10Z, gameNumber 2);
both status Scheduled, startTimeTBD false, doubleHeader flag "S" (split doubleheader);
identities distinct and stable; game 1 is the makeup of the postponed May 9 game.
Published probable starters: TB game 1 Griffin Jax; all others not yet listed.
No paid odds or source-readiness endpoint was called; final drill eligibility is NOT
determined today (screening happens at drill time per runbook section 2).

Fallback rule (reconfirmed valid): use both doubleheader games only if BOTH qualify at
screening; otherwise defer the doubleheader experiment and select one or two ordinary
eligible MLB games; doubleheader unavailability is NOT an RC failure.

## 9. startup/shutdown rehearsal (phase 7 -- health-check and shutdown only)

Startup per runbook section 1 (Docker Desktop -> `docker start devcore-sql` (~15s) ->
`dotnet run` in DevCore.Api -> uvicorn WITHOUT canary flag): SQL container Up (1433);
API listening 5007 (pid verified = DevCore.Api); agent-service listening 8000 (pid
verified = python; also binds gRPC 50051 as documented); `GET /health` and
`GET /api/ping` both 200 `{"status":"ok"}`; 4200/4201 stayed free (console not
required by the runbook for this rehearsal). Zero product runs created; zero paid or
external sports calls (logs show route registration + local pings only).

Shutdown per runbook section 8: agent-service stopped; `scripts/stop-platform-api.ps1`
**exit 0** ("port 5007 released, no api process remains"); `docker stop devcore-sql`
-> Exited; Docker Desktop quit; final verification: 5007/8000/4200/4201/1433/50051 all
free, zero stale project processes, docker daemon down -- runtime returned to the cold
state in which it was found. No unrelated process touched.

Deviations (recorded, non-blocking): (a) uvicorn and Docker Desktop were stopped via
Stop-Process on their exact pids instead of interactive ctrl-c / tray quit (this
harness cannot send ctrl-c; the runbook's own escalation path); (b) build emitted the
pre-existing NU1903 advisory warning for Microsoft.OpenApi 2.0.0 -- a dependency
advisory, not a runtime defect; not release-critical for the local concierge pilot; no
code change made (engineering follow-up would need its own WI); (c) Angular was never
started, so "Angular stopped" is satisfied vacuously.

## 10. friday execution packet (rc gate 1, 2026-07-17)

Authoritative documents (do not duplicate; execute from them):
runbook `06 Execution/plans/v1-concierge-operations-runbook-v1.md` (sections 1-4, 7, 8);
drill package `06 Execution/plans/v1-rc-drill-package-v1.md` (script steps 1-8);
ledger template `06 Execution/plans/v1-delivery-ledger-template-v1.md`.
Local checklist mirror: `rc-drill-2026-07-17\command-checklist.md`.

- **hashes:** dai main == origin/main == `85a8831`; dai-vault == origin/main at the
  commit carrying this record (re-verify both at drill open, runbook 1.1).
- **configuration:** section 6 checklist -- all present as of Gate 0.
- **stripe:** operator supplies the approved TEST-MODE link before/at drill; otherwise
  execute all technical steps and record the entitlement criterion as CONDITIONAL.
- **candidates:** TB@BOS DH 824766 + 824737 (section 8); fallback rule applies.
- **hard caps:** max 2 paid model calls, max 2 created runs, max 2 screens per
  candidate (incl. one R9 re-screen), zero reconciliation writes under gate 1.
- **execution order:**
  1. opening checks + startup (runbook 1; rehearsed, section 9 above)
  2. stripe test-mode dry-run (runbook 7) IF link provided -- ledger `entitlement=test`
  3. screening (runbook 2): statsapi identity check, then source-readiness per
     candidate WITH explicit gamePk; apply fallback rule
  4. generation (runbook 3): shell-scoped `DAI_MLB_REGISTRY_PROMPT_CANARY=1`; canary-
     first POST with explicit gamePk; verify prompt-trace/calibration-row/priced
     metering per run
  5. deliberate duplicate step: re-POST run 1's game once -> expect 409 +
     existingAgentRunId, zero spend
  6. forced source-outage scenario (drill package step 6): invalid odds key
     shell-scoped -> observable failure -> recover per R4/R9 -> one re-screen
  7. run 2 if authorized and eligible
  8. brief render + double-fetch determinism check; simulated delivery to
     `rc-drill-2026-07-17\simulated-delivery\` + operator's own address only
  9. shutdown to cold (runbook 8; rehearsed)
- **stop conditions:** `rc-drill-2026-07-17\stop-conditions-checklist.md` (cap breach,
  identity mismatch, guard FAIL, fallback, unpriced warning, 2 consecutive failures,
  unrecoverable-from-runbook anything).
- **gate 1 report path:** working file `rc-drill-2026-07-17\rc-record-working-v1.md`
  -> committed as `06 Execution/reports/v1-rc-drill-record-2026-07-17-v1.md`.
- **gate 2 preflight boundary:** next day, READ-ONLY finals guard + strict preflight +
  feed/live re-verify only; reconciliation writes require their own authorization
  (skeleton: `rc-drill-2026-07-17\gate2-preflight-skeleton.md`). After gate 2: recap
  verification -> final RC verdict -> EXPLICIT OPERATOR DECISION. A passing RC verdict
  authorizes no outreach, buyer contact, payment collection, cloud, multisport,
  automation, or feature work.

## 11. boundary confirmation

dai untouched (verified by diff before commit); no branch created; no WI minted; no
candidate promoted; no paid model call; no paid sports-source call; no sports run; no
run-creation endpoint called; no pending product rows; no reconciliation; no DB write;
no RC drill or forced-outage execution; no Stripe access or transaction; no prospect/
buyer contact, samples, or outreach; no prompt/model/scoring/confidence/routing/
calibration/settlement/schema change; csproj phantom untouched; both intentionally
untracked vault files untouched; posture remains no-spend; no commercial, spend,
capture, reconciliation, or payment authorization inferred.
