# platform execution sequence

**date:** 2026-04-13
**companion to:** `02 Platform/architecture/next-platform-architecture-plan.md`
**scope:** auth, identity, service communication, streaming, observability — sequenced for the next few milestones
**not covered here:** sports analysis pipeline milestones (see `sports-v1-roadmap.md`)

---

## platform rule: single tenant per deployment in v1

All users of a given deployment share one platform tenant. There is no per-user or per-org tenant provisioning at sign-up. `DevProvisionController` provisions one tenant per deployment; consumer sign-up does not create new tenants. This is not a limitation to work around — it is the correct shape for v1. Revisit only when multi-workspace isolation is a real product requirement.

---

## now
*no blockers — can start today*

- **angular routing** — milestone 1 of the angular plan (route structure, feature shells) has no platform dependencies
- **dev stack** — smoke tests (`test-sports-dev.ps1`) should pass on every session start; treat a failing smoke test as a blocker before other work
- **`useStubApi`** — leave `true` in `environment.ts` (production) until milestone 1 of the sports analysis plan is validated in a real deployment; do not flip it prematurely
- **RenameOaklandToSacramento migration** — apply to the running Docker DB before any real deployment; the `OddsScheduleClient._nameMap` covers both names in the meantime

**completed in this session (2026-04-13):**
- ~~sports analysis milestone 2~~ — structured factor output (`"category: observation"` shape) verified for NBA and MLB
- ~~observability baseline~~ — `ILogger<T>` in `FastApiClient` and `AgentRunService`, `X-Correlation-Id` header flow .NET → FastAPI, FastAPI logs correlation ID + matchup details on every request, FastAPI error body logged before throw
- ~~Odds API schedule integration~~ — `OddsScheduleClient` live; NBA and MLB real matchup dates verified; Athletics normalization confirmed (`"Athletics"` regardless of franchise name); `useStubSchedule` flipped to `false` in development

---

## next
*portal and config decisions must precede code changes — do not start code work until the named prerequisite is confirmed*

### step 1 — portal work (no code)
1. provision the Entra External ID tenant in the Azure portal
2. configure Google, Apple, and Microsoft personal account as federated identity providers in that tenant
3. register the `.NET` API as an app registration; set the audience (`api://{app-id}`)
4. register the Angular SPA; set redirect URIs for `localhost:4201` and the production origin

Nothing below this line starts until all four portal steps are done and a test token can be acquired.

### step 2 — auth generalization (after portal work is confirmed)
1. **update `Program.cs` authority** — change JWT Bearer authority from `login.microsoftonline.com/{tenantId}/v2.0` to the External ID tenant authority (`{tenant-name}.ciamlogin.com/{tenant-id}/v2.0`); no other middleware changes
2. **generalize `IdentityResolver`** (`DevCore.Api/Identity/IdentityResolver.cs:50`)
   - read `idp` claim; normalize to provider slug: `"google.com"` → `"google"`, `"apple.com"` → `"apple"`, `"MSA"` → `"microsoft"`, absent → `"entra"`
   - `Subject` stays `oid` — stable regardless of social provider
   - no schema changes; `UserIdentity` unique index already handles multi-provider
3. **generalize `DevProvisionController`** (`DevCore.Api/Controllers/Dev/DevProvisionController.cs:117`)
   - same `idp`-based normalization; drop `tid`-based tenant splitting — External ID tokens have no `tid`; one tenant per deployment per the platform rule above
4. **add `[Authorize]` to `AgentRunsController`** — only after steps 1–3 are tested end-to-end with a real token; not before
5. **MSAL Angular config** — add `MsalInterceptor` in `app.config.ts`; interceptor attaches bearer tokens to `.NET` API calls; no other Angular changes required

### step 3 — odds api ✓ complete (2026-04-13)
- schedule lookup implemented in .NET (`OddsScheduleClient`), not Python — schedule data is reference data, does not feed prompts, no AI processing required
- team name normalization confirmed: `"Oakland Athletics"` / `"Sacramento Athletics"` → `"Athletics"`; covered in `_nameMap`; no Python changes needed

---

## streaming ladder
*follow this in order — do not jump ahead*

| step | what | when |
|---|---|---|
| 1 — now | fake cycling messages via `interval(2500)` → `analysisPhase` signal in `app.ts` | already in place |
| 2 — if needed | client polls `GET /api/agent-runs/{runId}/status` every 2–3s; Angular replaces `interval` with a polling observable | if real-time phase feedback becomes a user problem before the pipeline is stable |
| 3 — later | authenticated `fetch + ReadableStream` SSE from `.NET` `/api/agent-runs/{runId}/events`; `[Authorize]` on the endpoint; events carry phase labels only — no internals | when pipeline is stable and real-time progress is a confirmed product requirement; note: use `fetch`, not native `EventSource` — `EventSource` cannot set `Authorization` header |
| 4 — much later | gRPC server streaming from Python → .NET if internal service complexity or multiple Python services justify it | only if condition is real, not anticipated; port 50051 is scaffolded |

The `narration` computed in Angular is unchanged across all four steps — only the observable driving `analysisPhase` changes.

---

## later
*explicitly deferred — named trigger for each*

| item | trigger to revisit |
|---|---|
| jit provisioning in `IdentityResolver` | when auth is end-to-end tested and consumer sign-up is a real need |
| `GamesService` + game list in Angular | when `GET /api/sports/{slug}/games` exists in .NET |
| brief-ready status for game list | client-side polling every 60s is correct for v1; upgrade to SSE when latency is a real user problem |
| stripe webhook + `TierGuard` | when billing is a concrete near-term milestone |
| azure functions (scheduled triggers) | when brief generation works reliably on demand |
| account feature in Angular | when delivery preferences exist to configure |
| `useStubApi: true` removal | after sports analysis milestone 1 is validated in a real deployment |

---

## do not build yet

These are valid ideas. None of them are the right next thing. Build any of these only when a specific, concrete problem in production makes them the clear answer.

| item | reason to wait |
|---|---|
| Kubernetes | no scale problem exists; adds operational complexity with no current benefit |
| vector database | no RAG use case in the active roadmap; adds infra dependency for a speculative feature |
| RAG pipeline | current sports analysis does not require retrieval; revisit when factual grounding of real-time data is the bottleneck |
| Redis | no caching bottleneck exists; SQL Server with proper indexing is sufficient for current read patterns |
| Service Bus / message queue | the pipeline is synchronous by design; async brief generation adds failure modes without current benefit |
| true async brief pipeline | the current request-response path works; async only makes sense when scheduled generation and delivery are both live |
| broad gRPC rollout | there is one Python service and one .NET service; gRPC adds protocol complexity for no current gain |
| multi-provider auth coding before External ID tenant provisioned | writing `IdentityResolver` generalization without a real token to test against creates untestable code; portal work must come first |

---

## blocked by decisions
*nothing can move on these until a named decision is made*

| decision | who acts | what unblocks |
|---|---|---|
| provision the Entra External ID tenant and configure social providers | azure portal — founder action | all of "next / step 2" above |
| auth gate timing for `AgentRunsController` | product decision | whether `[Authorize]` is a pre-launch requirement or can wait until first external deployment |
| jit vs manual provisioning path for consumer sign-in | product decision | shapes when `IdentityResolver` is generalized and `DevProvisionController` is simplified |
| `useStubApi: true` flip | schedule as a named milestone | flip only after a real deployment is validated; must be a deliberate act, not a forgotten default |
