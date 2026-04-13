# next platform architecture plan

**date:** 2026-04-12
**scope:** customer auth, multi-provider identity, service communication, streaming, external integrations
**grounded in:** actual code in `platform/dotnet/`, `services/agent-service/`, `apps/sports-app/`
**status:** forward plan — distinguishes what is real today from what is next

---

## 1. customer authentication strategy: entra external id

### what is real today

`Program.cs` has JWT Bearer wired to the workforce tenant endpoint:

```
https://login.microsoftonline.com/{tenantId}/v2.0
```

`AgentRunsController` and `SportsReferenceController` have no `[Authorize]` attribute. `IdentityResolver` has a dev bypass via `Dev:EnableBypassAuth`. The auth scaffolding is in place but no real users can sign in yet.

### recommended direction: entra external id

Entra External ID (the new consumer-facing identity product, not the older B2C flows) is the right fit. It acts as a single JWT issuer that federates Google, Apple, and Microsoft personal account sign-in. From .NET's perspective the change is one configuration value — the authority URL changes from the workforce tenant format to the External ID tenant format:

```
https://{tenant-name}.ciamlogin.com/{tenant-id}/v2.0
```

JWT Bearer middleware behavior is identical. Token validation, audience check, and claim extraction all work the same way. The social providers (Google, Apple, Microsoft) are configured in the Entra External ID portal — no code changes when a provider is added or removed.

On the Angular side, MSAL Angular is configured with the External ID authority and user flow. Sign-in, token acquisition, and silent renewal are handled by MSAL. An HTTP interceptor attaches `Authorization: Bearer {token}` to calls to the .NET API — this is the only Angular-side addition needed.

### why not auth0 / okta

Both are valid but add a paid third-party dependency for something Azure already provides under the existing Azure subscription. The platform is already on Azure. Entra External ID pricing is generous at the volumes dai-v1 will see.

### what this does not solve

- tier enforcement — that is Stripe's job
- tenant provisioning — that stays in `DevProvisionController` flow, generalized for External ID tokens
- internal service-to-service auth — .NET→Python is on a private network; no token needed there in v1

---

## 2. how google, apple, and microsoft sign-in map into user / useridentity

### the identity model (no schema changes needed)

`UserIdentity` has `(Provider, Issuer, Subject)` with a unique index on `(TenantKey, Provider, Issuer, Subject)`. The `Provider` field comment already lists "google" as an example. No migrations are needed for multi-provider support.

### how external id tokens map to the tuple

External ID issues all tokens from its own authority regardless of which social provider the user authenticated through. Every token contains:

| claim | meaning |
|---|---|
| `oid` | the user's object id in the External ID tenant — stable, unique per user |
| `iss` | the External ID tenant issuer URL — stable |
| `idp` | which social provider was used — `"google.com"`, `"apple.com"`, `"MSA"` (microsoft personal), or absent for direct sign-in |

The `UserIdentity` row for a user who signs in with Google:

```
Provider = "google"
Issuer   = "https://{tenant}.ciamlogin.com/{tenant-id}/v2.0"
Subject  = "{oid}"  (the oid from the External ID token)
```

The `oid` is the subject in all cases — it is External ID's stable identifier for the user. The `idp` claim is used only to populate the `Provider` slug. Two users who sign in with different social providers have different `oid` values and therefore different `UserIdentity` rows. Account linking (one person, multiple sign-in methods) is explicitly deferred.

### what changes in `IdentityResolver`

`ResolveFromPrincipalAsync` currently hardcodes `provider = "entra"` and reads `oid`. The change is:

1. read the `idp` claim
2. normalize it to a stable provider slug:
   - `"google.com"` → `"google"`
   - `"apple.com"` → `"apple"`
   - `"MSA"` → `"microsoft"`
   - absent or unknown → `"entra"` (direct External ID sign-in)
3. `Subject` stays `oid` — it is stable regardless of social provider

The rest of the resolver — tenant lookup, disabled check, `TenantKey`/`UserKey` resolution — is unchanged.

### jit provisioning

Today `IdentityResolver` throws `UnauthorizedAccessException("user not provisioned")` when no `UserIdentity` row exists for a valid JWT. That is correct during dev while `DevProvisionController` is the provisioning path.

For consumer sign-in, just-in-time provisioning is the right pattern: if a valid External ID JWT arrives and no `UserIdentity` row exists, the resolver creates the `Tenant`, `User`, and `UserIdentity` rows in one transaction. This removes the provisioning step as a manual prerequisite for new users.

JIT provisioning should be implemented when auth is turned on, not before. The dev bypass path is unaffected.

### `DevProvisionController` changes

Same `idp`-based normalization logic as `IdentityResolver`. Currently hardcoded to `provider = "entra"` and reads `oid` and `tid`. The `tid` claim is a workforce tenant concept — External ID tokens do not contain it. The tenant boundary for External ID users is either:

- the External ID tenant itself (all users share one platform tenant — simple, works for v1)
- a tenant provisioned at sign-up (multi-tenant SaaS — adds a sign-up flow, defer)

For v1: all External ID users provision into a single platform tenant per deployment. `DevProvisionController` can be simplified to drop the `tid`-based tenant splitting.

---

## 3. frontend to backend communication

**keep as REST. no changes.**

Angular → `.NET` via `HttpClient` with MSAL interceptor is the standard, well-understood pattern for SPAs calling protected APIs. The sports-app already calls `POST /api/agent-runs`, `GET /api/sports`, `GET /api/sports/{slug}/teams`, and `GET /api/sports/{slug}/matchup-dates`. All of these stay as REST.

When auth is enabled:
- add `MsalInterceptor` in `app.config.ts` to attach bearer tokens to `.NET` API calls
- add `[Authorize]` to `AgentRunsController` — the first gated endpoint
- `SportsReferenceController` can stay public (reference data) or be gated — decide at launch

No WebSockets, no gRPC, no GraphQL. REST is the right choice for the Angular→.NET boundary.

---

## 4. .net to python communication

**keep as HTTP. no changes.**

`FastApiClient` calls `POST http://127.0.0.1:8000/api/sports/analyze` with a 90-second timeout. This is a private loopback call — the FastAPI service is not internet-exposed. The HTTP contract (`SportsAnalysisRequest` / `SportsAnalysisResponse`) is stable and mirrored across three files. There is no operational reason to change this path.

The Python side has port 50051 scaffolded for gRPC. Ignore it until there is a concrete use case (see section 5).

---

## 5. where http stays and where grpc makes sense later

### keep HTTP for

- Angular → .NET (REST is the right transport for browser clients)
- .NET → FastAPI for sports analysis (single call, request-response, working today)
- .NET → Azure Functions (HTTP trigger is the correct invocation pattern)
- .NET → SendGrid (HTTP client, vendor SDK)
- Stripe → .NET (webhook, HTTP POST)

### grpc becomes worth adopting when

Two specific conditions justify introducing gRPC:

**condition A — multiple Python services talking to each other**
If the sports analysis pipeline splits into separate processes (e.g. a `collector-service` and an `evaluator-service`), gRPC between them is cleaner than REST between Python services. Strongly typed proto contracts, smaller payloads, and native streaming all add real value in an internal Python-to-Python boundary.

**condition B — streaming progress from Python back to .NET**
If FastAPI needs to stream progress events to .NET as an analysis runs (rather than .NET polling), gRPC server streaming is the right transport. The alternative — .NET polling FastAPI — works but adds latency and is architecturally messier than a real stream.

### grpc planning when the time comes

- proto files live in `contracts/proto/` at the monorepo root
- .NET consumes them via `Grpc.Tools` (generated client stubs)
- Python consumes them via `grpcio-tools` (generated stubs)
- the Python port 50051 scaffold is already in place
- do not introduce gRPC until one of the two conditions above is real, not anticipated

**risk:** gRPC debugging is harder than HTTP/JSON. Use it where streaming or internal contract rigidity clearly adds value. Do not use it to replace any HTTP surface that is working.

---

## 6. progress streaming: sse first, with security risks called out

### the designed seam (already in place)

`app.ts` drives in-flight narration messages via `interval(2500)` writing to `analysisPhase` signal. The `narration` computed reads `analysisPhase()` — it does not care what drives it. Replacing `interval(2500)` with a real event stream observable is the only change needed in Angular. The `narration` computed, the template, and the UI are all unaffected.

### recommended approach: fetch + readablestream (not native eventsource)

SSE is the right first choice for progress streaming — it works over plain HTTP, needs no new infrastructure, and Angular handles it naturally with an Observable wrapper. However, native browser `EventSource` cannot set custom request headers, which means `Authorization: Bearer` cannot be attached. This is a hard limitation for a protected API.

**use `fetch()` with `ReadableStream` instead of `EventSource`**. The `fetch` API supports custom headers, works with MSAL-acquired tokens, and produces a `ReadableStream` that can be consumed as a line-by-line observable. Angular wraps it cleanly.

### .net side

A new endpoint `GET /api/agent-runs/{runId}/events` returns `text/event-stream`. It holds the connection open and forwards progress events as the analysis runs. It requires `[Authorize]` — the token is validated on the initial `fetch` request using the standard JWT Bearer middleware.

### security risks and mitigations

| risk | severity | mitigation |
|---|---|---|
| unauthenticated stream access | high | `[Authorize]` on the stream endpoint; validated on initial fetch, not per-event |
| event content leaking internals | high | events carry only phase label strings — no stack traces, no prompt content, no chain-of-thought |
| connection held open after navigation | medium | Angular component closes the `fetch` `AbortController` in `ngOnDestroy` |
| server connection exhaustion | medium | enforce a max connection timeout (e.g. 120s); close the stream on analysis completion or error regardless |
| MSAL token expiry mid-stream | low | in-flight streams close on token expiry; Angular retries from the result polling fallback |
| FastAPI progress not available yet | n/a | .NET can simulate progress with its own phase advancement until Python sends real events |

### fallback: polling

If SSE adds complexity before it adds clear value, `.NET` can expose `GET /api/agent-runs/{runId}/status` and Angular polls it every 2–3 seconds. Same `analysisPhase` signal seam, simpler implementation. Only adopt SSE when real-time progress is a product requirement, not a nice-to-have.

---

## 7. azure functions, sendgrid, and stripe webhook roles

Each of these has one job. They should not do more than that job.

### azure functions: dumb timers only

Functions fire on a timer schedule and call a `.NET` internal endpoint to trigger brief generation. No business logic lives in Functions. The Function looks like:

```
TimerTrigger → POST /api/internal/trigger-briefs?sport=nfl&date={today}
             → .NET validates the request and dispatches to FastAPI
             → FastAPI runs the analysis
             → .NET persists the brief
```

The internal endpoint is not internet-exposed. It uses a shared secret header, not a bearer token. Do not create Azure Functions until the sports analysis pipeline is stable and brief generation is working on demand — scheduling an unstable pipeline does not help.

### sendgrid: delivery only

After a brief is persisted, `.NET` calls SendGrid to deliver it. Template rendering, personalization, and recipient resolution all happen in `.NET` before the call. `DeliveryService` in `.NET` owns the send. SendGrid is a transport, not a logic layer.

### stripe: billing source of truth, .net is the enforcement layer

The Stripe webhook hits `POST /api/webhooks/stripe` — a dedicated controller in `.NET` with:
- webhook signature verification (Stripe-Signature header, endpoint secret)
- no `[Authorize]` — this is a public endpoint, authenticated by the signature only
- writes subscription state to a `Subscription` table scoped to `TenantKey`

Tier enforcement reads from that table. No live Stripe calls on the request path. The `TierGuard` (future) queries `.NET`'s own DB, not Stripe's API.

Do not scaffold Stripe webhook infrastructure until billing is a real near-term milestone.

---

## 8. what should stay thin now

| area | current state | hold the line |
|---|---|---|
| auth on sports endpoints | no `[Authorize]`, dev bypass in `IdentityResolver` | do not add `[Authorize]` to `AgentRunsController` until the External ID tenant is provisioned and tested end-to-end |
| External ID tenant | not provisioned | this is an Azure portal task, not a code task — must happen before any real sign-in can be tested |
| `GamesService` in Angular | does not exist | keep stub until `GET /api/sports/{slug}/games` endpoint exists in .NET |
| tier enforcement | no Stripe, no `TierGuard` | do not scaffold `TierGuard` or Stripe webhook infrastructure until billing is a real near-term milestone |
| azure functions | none created | do not create Function Apps until brief generation works reliably on demand |
| gRPC | port 50051 scaffolded, unused | do not generate proto stubs until there is a second Python service or a real streaming requirement |
| account feature in Angular | empty shell component | leave it empty until delivery preferences exist to configure |
| SSE streaming | `interval(2500)` seam in place | do not implement SSE until the sports analysis pipeline is stable and real-time progress is a product need |
| `useStubApi` in production env | `true` in `environment.ts` | flip to `false` only after milestone 1 of the sports analysis plan is validated in a real deployment |

---

## 9. key decisions that must be made next

These are not design questions — they are concrete decisions that block or shape upcoming work. Each one has a named owner action.

**decision 1 — provision the External ID tenant**
An Azure portal action. Until an External ID tenant exists, no social sign-in can be tested and `IdentityResolver` generalization has no validation target. This is the prerequisite for all auth work.

**decision 2 — auth gate timing for `AgentRunsController`**
Currently any caller can create `AgentRun` rows. The question is whether to add `[Authorize]` before external users can reach the sports-app, or whether to treat the sports-app as internal-only (and therefore unprotected) until a real launch. If the app stays on `localhost:4201` and isn't deployed, this is not urgent. Once there is a deployment URL, it is.

**decision 3 — JIT provisioning vs manual provisioning for consumer sign-in**
`DevProvisionController` is a manual provisioning path (dev only). Consumer sign-in needs JIT: first JWT → auto-provision Tenant, User, UserIdentity. This decision shapes when `IdentityResolver` is generalized.

**decision 4 — team name format for odds API (milestone 3 of sports analysis plan)**
The odds API returns full team names (`"Kansas City Chiefs"`). The Teams table has full names. If they match exactly, milestone 3 is simpler. If they differ, a normalization map is needed in `sports_analyzer.py` before milestone 3 begins. Confirm before building `odds_client.py`.

**decision 5 — brief-ready status delivery for the game list**
The future `GamesService` in Angular needs to know when a brief transitions from `pending` to `ready`. Options:
- client-side polling on an interval (simplest — `GamesService` polls `GET /api/sports/{slug}/games` every 60s)
- SSE from .NET pushed when a brief status changes (lower latency, same fetch+ReadableStream pattern)

Polling is correct for v1. SSE is an upgrade once the polling latency is a real user problem.

**decision 6 — remove `useStubApi: true` from production environment**
This flag is the only thing preventing real API calls in a production build. It should be removed (not just flipped) as a named milestone once the sports analysis path is validated under real conditions. Do not forget it exists.

---

## what this plan does not touch

- `User` / `Tenant` / `UserIdentity` schema — no migrations needed for multi-provider auth
- `AgentRun` persistence model — `InputJson` / `OutputJson` shape is an internal concern
- `.NET` → FastAPI HTTP contract — stable and sufficient through milestone 4 of the sports analysis plan
- Angular component tree — Angular plan milestones 1–3 are unaffected by auth or streaming decisions
- FastAPI sports analyzer — milestone progression is independent of platform auth decisions
