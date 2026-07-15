---
title: "V1 Concierge Operations Runbook v1"
type: "plan"
date: "2026-07-14"
status: "active"
project: "DAI"
slice: "WI-0013 Pilot Operations Hardening v1"
repos:
  dai: "referenced (no change by this doc)"
  dai-vault: "docs-only"
tags:
  - runbook
  - operations
  - release
related:
  - "04 Products/sports-v1/v1-release-definition-and-scope-freeze-v1.md"
  - "06 Execution/plans/v1-delivery-ledger-template-v1.md"
  - "06 Execution/plans/v1-rc-drill-package-v1.md"
  - "02 Platform/system-development/work-items/WI-0013-pilot-ops-hardening.md"
---

# v1 concierge operations runbook v1

The complete daily V1 pilot workflow, executable by an operator who did not implement the
system. One buyer-facing promise: a pregame decision brief before first pitch and a settled
recap the next morning, delivered manually by email. Paid actions in this runbook run ONLY
under an explicit current authorization -- this document never grants one.

Binding caps per delivery day unless the day's authorization says otherwise: **max 4 runs,
max 4 paid model calls, stop on the first hard failure you cannot recover with this
runbook.**

## 1. opening checks (cold start)

1. Repos: `git -C dai fetch origin` then verify `git -C dai rev-parse main origin/main`
   are EQUAL and match the approved release commit in the current RC/pilot record. Same
   for dai-vault. STOP if they differ -- do not operate an unapproved tree.
2. Working tree: only the documented `DevCore.Data.csproj` line-ending phantom may be
   modified; the two documented untracked vault files stay untouched.
3. Runtime cold check: ports 5007/8000/4200/4201/1433 free; `docker ps -a` shows
   `devcore-sql` Exited; no `DevCore.Api`/`dotnet`/`uvicorn` processes. If something is
   already running, follow section 8 shutdown first.
4. Secrets/config presence (NEVER print values): `platform/dotnet/DevCore.Api/
   appsettings.Development.json` exists (SQL, odds key, Entra dev values);
   `services/agent-service/.env` exists (OPENAI_API_KEY). Registry route: confirm
   `DAI_MLB_REGISTRY_PROMPT_CANARY` is NOT set in `.env` (it is applied process-scoped at
   generation only, section 3).
5. Start order: Docker Desktop -> `docker start devcore-sql` (wait ~15s) ->
   `dotnet run` in `platform/dotnet/DevCore.Api` (default profile; expect 5007) ->
   agent-service ONLY on a generation day: `.venv\Scripts\python.exe -m uvicorn main:app
   --port 8000` from `services/agent-service` (no --reload) -> sports-app only if the
   operator console is needed: `npm start` in `apps/sports-app`.
6. Health: `GET http://localhost:5007/health` -> 200 {"status":"ok"}; on generation days
   `GET http://localhost:8000/api/ping` -> 200 {"status":"ok"} (the agent-service has no
   /health route -- /api/ping is its liveness probe).
7. Authorization check: read the current day's operator authorization (paid caps, named
   candidates). No authorization document = read-only day, no generation, no settlement.

## 2. slate screening (free, read-only)

1. Upcoming games: statsapi schedule
   (`https://statsapi.mlb.com/api/v1/schedule?sportId=1&date=YYYY-MM-DD`) or the analyzer
   picker. Verify each candidate's identity: teams, date, **gamePk**, doubleHeader flag.
   For a doubleheader date, record BOTH gamePks; each game is captured only with its
   explicit gamePk (never inferred from order or pk magnitude).
2. Source readiness per candidate: `GET /api/agent-runs/source-readiness?...` (pass the
   explicit gamePk on DH dates). Classify: ELIGIBLE (identity matched + starter enriched +
   market backed_depth) / THIN (partial context) / NO-DATA. Only ELIGIBLE games are
   generation candidates; each readiness screen burns one odds-api call -- do not loop.
3. Duplicate check: `GET /api/agent-runs/recent` or
   `GET /api/agent-runs/prompt-route-calibration/rows` -- confirm no active run already
   exists for a candidate. The API now enforces this with a 409 at creation (WI-0013),
   but screening first avoids burning a request.
4. Record in the delivery ledger: selected gamePks with reasons; skipped candidates with
   reasons (thin data, duplicate, cap reached).

## 3. generation (PAID -- explicit authorization required)

1. Confirm today's authorization: named candidates, run cap, spend cap.
2. Start agent-service with the registry route process-scoped:
   set `DAI_MLB_REGISTRY_PROMPT_CANARY=1` (and the allowlist variable if the
   authorization names one) IN THE SHELL ONLY, then start uvicorn. `.env` is never
   edited; the flag dies with the process.
3. Create runs canary-first (first game fully verified before the rest):
   `POST /api/agent-runs` body
   `{"runType":"sports.matchup.analysis","agentProfileId":null,
     "input":{"competition":"mlb","homeTeam":"...","awayTeam":"...",
              "gameDate":"YYYY-MM-DD","gamePk":NNNNNN}}`
   (gamePk REQUIRED on doubleheader dates; recommended always once screened.)
4. Duplicate 409: see recovery R1. Failed run: R5. Pending that never completes: R5.
5. Verify EVERY created run before the next one:
   - `GET /{id}/prompt-trace`: promptSource=registry, selectedPromptVersion=v2, recipe
     `...backed_depth.v2`, attribution complete, no fallback;
   - `/api/agent-runs/prompt-route-calibration/rows` row: externalGameId == the requested
     gamePk, guard Pass, active;
   - agent-service cost log line: `pricingStatus: "priced"` with a real
     estimatedTotalCost (an UNPRICED warning = R3, stop generation);
   - record model-call count, odds calls, cost, run id in the ledger.
6. STOP conditions: cap reached; any identity mismatch; any guard FAIL; any fallback;
   any unpriced-model warning; two consecutive failed runs.

## 4. pregame delivery (free)

1. `GET /api/agent-runs/{id}/brief` -- confirm: identity block matches the intended game
   (teams, date, gamePk), hasPosition/no-position correct, strengthBand present, market
   context line reads sensibly, NO numeric confidence anywhere.
2. `GET /api/agent-runs/{id}/brief/markdown` -- fetch twice; byte-identical expected.
3. Paste the markdown into the buyer email template and send manually. Deliver BEFORE
   first pitch or do not deliver (record a delivery failure instead, R13).
4. Ledger: brief generated ts, delivered ts, delivery status.

## 5. settlement (free reads; WRITES need their own authorization)

1. Next morning: finals guard `scripts/dev/sports/check-settlement-finals.ps1` on the
   day's gamePks -- proceed only on READY for the full cohort (partial cohorts are never
   settlement-ready).
2. Strict preflight: `scripts/dev/sports/preflight-settlement.ps1 -Competition mlb
   -GamePks ... -RequireUnreconciled -FailOnWarnings` -> exit 0.
3. Re-verify score + identity per game from statsapi `feed/live` immediately before each
   write.
4. With the day's settlement authorization: identity `POST /api/agent-runs/reconcile`
   per game with full residue (source, sourceRef, notes). Expect SingleMatch on the
   intended run. MultipleMatches/NoMatch -> STOP (R11).
5. Verify: outcome + evaluation exist (`GET /{id}/evaluation`), exactly N rows changed,
   0 duplicate outcomes, dup-active 0. Re-running a settled write must 409 (idempotency).
6. Postponed/cancelled events: never settle against a replay -- follow the 823357
   precedent (operator exclusion decision, its own authorization; R12).

## 6. postgame delivery (free)

1. `GET /api/agent-runs/{id}/recap` -- verify recapState is the expected one:
   settled_evaluated for a settled directional read (score + persisted result);
   no_position (score shown, explicitly not scored); excluded ("No result -- event not
   evaluated."); not_settled / no_result / settled_not_evaluated as honest intermediate
   states. An `inconsistent` state -> STOP, R7.
2. `GET /api/agent-runs/{id}/recap/markdown` -- deterministic; verify score and
   Result line; no numeric confidence; no aggregate language.
3. Send manually; update ledger (recap delivered ts, buyer feedback note when received).

## 7. payment-link and entitlement (manual; stripe = truth)

- Pilot entitlement = a verified Stripe receipt. Before ANY delivery: check the ledger
  row for the buyer shows entitlement `paid` (or `test` during the RC dry-run). Absent or
  unclear entitlement -> WITHHOLD delivery, record, contact the buyer.
- RC dry-run procedure (required by the RC gate, test-mode only): create/select the
  approved Stripe payment link; execute an EXPLICITLY MARKED test transaction (Stripe
  test mode); record `entitlement=test` against the pilot ALIAS in the ledger; the
  receipt id and the buyer email go ONLY in the private, uncommitted alias-mapping note
  (never in a committed artifact -- ledger privacy rule); confirm the receipt email
  matches the delivery email in that private note; then void/refund per Stripe test
  conventions. No Stripe code, webhook, or checkout page exists or is authorized.
- Refunds / failed payments / mismatched emails: resolve manually with the buyer;
  deliveries pause until the ledger row is consistent again (R14).

## 8. shutdown (return to cold)

1. Stop sports-app (ctrl-c the ng serve terminal).
2. Stop agent-service (ctrl-c uvicorn; if elevated/stuck, Stop-Process by pid).
3. Stop the API with `scripts/stop-platform-api.ps1` (the corrected WI-0004 script:
   verifies port 5007 released AND no api host remains; exit 0 required; exit 2 = an
   unrelated process owns 5007 -- investigate, do not kill blindly; exit 3 = timeout, R10).
4. `docker stop devcore-sql`; quit Docker Desktop.
5. Verify: 5007/8000/4200/4201/1433 free; 0 docker processes; no DevCore.Api/dotnet/
   uvicorn remain. Never kill unrelated processes.

## 9. failure recovery (bounded; each path ends in a stop condition)

| id | failure | recovery | retry? | new auth? | paid call ok? | record |
|---|---|---|---|---|---|---|
| R1 | duplicate 409 on create | inspect existingAgentRunId; if it IS the intended run, use it (no new run); if a different game on a DH date, re-POST WITH the distinct gamePk -- this clears the block ONLY when the existing run also carries a known gamePk (screened per section 2/3); if the existing run is matchup-only, the guard stays fail-closed and excluding or settling it first needs ITS OWN authorization -- stop there | yes, once corrected | only for exclusion | no extra beyond authorized cap | 409 body + decision |
| R2 | model timeout / model error (failed run) | run row persists as failed (does not block retry); check agent-service log; retry ONCE within cap | once | no (within cap) | yes, counts against cap | run id + error |
| R3 | UNPRICED model warning in cost log | STOP generation; fix `model_metering.PRICING` is engineering (new WI) -- do not continue an unmetered day | no | yes (engineering) | no | model name + requestId |
| R4 | source outage / market not backed_depth | game is NOT eligible; skip or wait for odds to post; re-screen at most twice (odds-api cost) | re-screen x2 | no | screen calls only | readiness result |
| R5 | stale pending run | wait 5 min; if still pending, treat as failed: verify no artifact/no cost line, then proceed (pending rows block duplicates -- excluding a dead pending row needs its own authorization) | no | for exclusion | no | run id + timeline |
| R6 | identity mismatch (trace/row gamePk != intended) | STOP the day's generation; the run must not be delivered; exclusion + capture-retry both need fresh authorization | no | yes | no | both gamePks |
| R7 | prompt fallback / attribution failure / inconsistent recap state | STOP; deliver nothing from that run; diagnostics via /prompt-trace; engineering follow-up | no | yes | no | trace snapshot |
| R8 | brief/recap endpoint failure (500/404 on an existing run) | retry the GET once; then check API log; if persistent, deliver nothing and stop | once | no | no (free reads) | status + log line |
| R9 | starter-cache poisoning (all screens read "no starters") | WI-0005 lesson: restart DevCore.Api (clears IMemoryCache), re-screen serially with pacing | yes | no | screen calls only | before/after screens |
| R10 | API shutdown failure (script exit 2/3) | exit 2: identify the port owner, never kill unrelated processes; exit 3: Stop-Process the DevCore.Api/dotnet pid directly, re-run script to verify | yes | no | no | pids + exit codes |
| R11 | settlement precheck ambiguity (MultipleMatches/NoMatch) | STOP settlement for that game; per-run /outcome or exclusion decisions are operator calls needing their own authorization | no | yes | no | precheck output |
| R12 | reconciliation mismatch (score/identity disagree at write time) | do NOT write; re-read feed/live; if still mismatched, stop and investigate | no | yes | no | both readings |
| R13 | email delivery failure | resend once; if still failing, record failed delivery, notify buyer via alternate channel, do not regenerate the run | once | no | no | ledger failure note |
| R14 | Stripe/entitlement discrepancy | withhold delivery; reconcile receipt vs ledger vs buyer email manually; resume only when consistent | n/a | no | no | discrepancy + resolution |

Every recovery beyond these bounds = stop the day, record the state, request a fresh
operator decision.
