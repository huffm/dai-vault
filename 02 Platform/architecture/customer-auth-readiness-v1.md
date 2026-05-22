# customer auth readiness v1

**date:** 2026-05-22
**status:** auth readiness assessment. no auth implemented in this slice. docs only. no Azure resources created.
**scope:** the customer sign-in plan DAI commits to before the sports product is publicly exposed on Azure Container Apps: which identity provider, which sign-in methods, what registrations, what env config, and what must happen before the public edge goes live.

## what this document is

The Azure Container Apps Provisioning Plan v1 flagged customer auth as a gating manual blocker and noted that the .NET API's JWT authority is currently the workforce Entra shape, not the customer (External ID) shape. This document removes that ambiguity. It states the customer identity target plainly, inventories what is real in the code today versus what the auth slice must add, and defines the registrations, env vars, CORS, and redirect/logout URLs for both local dev and cloud. It does not implement auth and creates nothing in Azure.

It consolidates and supersedes the auth-specific guidance scattered across `next-platform-architecture-plan.md` (section 1-2, the original Entra External ID strategy, 2026-04-12), `cloud-deploy-readiness-v1.md`, and `azure-container-apps-provisioning-plan-v1.md` section 14. Where those imply or could be read as workforce-only Entra, this document is the correction: the customer path is Microsoft Entra External ID (CIAM), federating Google, Apple, Microsoft, and email.

Architecture guardrails preserved: Cognitive Protocol Runtime behavior, Tool Gateway governance, CognitiveProtocol persistence, and the local dev workflow are unchanged. Nothing here bypasses the Tool Gateway. No FastAPI prompt, Pydantic contract, `CognitiveProtocolBuilder` mapping, confidence rule, database schema, MCP, pgvector, Azure Function, or Kubernetes resource is touched. No secret is committed.

## 1. current auth implementation (verified in code)

**.NET API (`DevCore.Api`):**
- JWT Bearer is wired in `Program.cs:42-53` with authority `https://login.microsoftonline.com/{AzureAd:TenantId}/v2.0` (the workforce Entra v2 shape) and audience `AzureAd:Audience`. The authority host is hardcoded to `login.microsoftonline.com`; only the tenant id is configurable.
- `app.UseAuthentication()` then `app.UseAuthorization()` are in the pipeline (`Program.cs:145-146`).
- The sports endpoints are not gated: neither `AgentRunsController` nor `SportsReferenceController` carries `[Authorize]`. Only `AiController`, `ConversationsController`, and `DevProvisionController` (the chat/legacy and dev-provision surfaces) carry `[Authorize]`. `/health` is anonymous.
- `IdentityResolver.cs` resolves an authenticated principal by hardcoding `provider = "entra"`, reading the `iss` and `oid` claims, and looking up a matching `UserIdentity` row; it throws `UnauthorizedAccessException("user not provisioned")` when no row exists. When unauthenticated, it falls back to a dev bypass only if `env.IsDevelopment()` and `Dev:EnableBypassAuth` is true (`IdentityResolver.cs:40`), resolving a configured `Dev:TenantKey`/`Dev:UserKey` with provider `"dev"`.
- `DevProvisionController.cs` is the manual provisioning path: dev-only, gated by an `x-provision-key` shared secret, reads `iss`/`oid`/`tid` from the token, uses `tid` as the tenant boundary, and hardcodes `provider = "entra"`.
- `UserIdentity` (`DevCore.Domain/Chat/UserIdentity.cs`) is the identity tuple: `(Provider, Issuer, Subject)`, with `Provider` examples already including `google`. `Subject` is documented as "the oidc sub or the entra oid depending on provider". No schema change is needed for multi-provider support.

**Angular sports app (`apps/sports-app`):**
- No authentication today. `app.config.ts` provides only `provideRouter` and `provideHttpClient()`; there is no MSAL, no HTTP interceptor, no token acquisition, no login UI. A repo-wide search finds no `Authorization`/`Bearer`/`msal` usage in `apps/sports-app/src`.
- The client calls the .NET API with no bearer token. Today it works because the sports endpoints are ungated and dev bypass supplies identity server-side.
- MSAL packages do exist in the monorepo under the separate `prompt-portal` app (`devcore-ng`) `node_modules`; that is a different, internal app and is not the customer-facing precedent. The sports-app must add MSAL fresh.

**Net:** the auth scaffolding (.NET JWT middleware, the identity tuple, the strict resolver, a dev bypass) exists, but no real customer can sign in. The client sends no token, the sports endpoints are not gated, and the authority is the workforce shape.

## 2. target: microsoft entra external id (decision)

**The customer identity provider is Microsoft Entra External ID** (the consumer-facing CIAM product, not the older B2C tenant flows and not workforce Entra). This is confirmed, not reopened. Rationale, unchanged from `next-platform-architecture-plan.md` section 1:

- DAI is already on Azure; External ID is included under the existing subscription with generous pricing at v1 volumes. No new paid third-party identity dependency (Auth0/Okta/Clerk) is taken on.
- External ID is a single JWT issuer that federates Google, Apple, and Microsoft personal-account sign-in plus native email. From .NET's view it is one issuer; social providers are configured in the External ID portal with no code change to add or remove one.
- The existing `(Provider, Issuer, Subject)` identity model already fits (section 1 above and section 4 below). No migration.

**The one .NET code change this implies (deferred to the auth slice, see section 12):** the JWT authority must move from the workforce host to the External ID host:

```
workforce (today):     https://login.microsoftonline.com/{tenant-id}/v2.0
external id (target):   https://{tenant-subdomain}.ciamlogin.com/{tenant-id}/v2.0
```

JWT Bearer middleware behavior is otherwise identical: token validation, audience check, and claim extraction are unchanged. The change is one authority value, which should be made configurable rather than hardcoded (section 5).

## 3. supported customer sign-in methods

All four are delivered through the single External ID tenant. The application code does not branch per method; External ID issues one token shape regardless of how the user authenticated.

| method | how it is delivered | provider slug in `UserIdentity` |
|---|---|---|
| email (one-time passcode or password) | native External ID local account; recommend email one-time passcode (OTP) to avoid storing/managing passwords | `entra` (no `idp` claim on direct sign-in) |
| Google | Google federation configured in the External ID portal | `google` |
| Apple | Apple federation configured in the External ID portal | `apple` |
| Microsoft | Microsoft personal account (MSA) federation via External ID | `microsoft` |

**Email recommendation:** prefer email one-time passcode over email/password. OTP removes password storage, reset flows, and credential-stuffing surface, and External ID supports it natively. Email/password remains available in the portal if a password option is later required; it is a portal toggle, not a code change.

**Microsoft path:** delivered through External ID's Microsoft (MSA) federation, normalized to the `microsoft` slug. This is the consumer Microsoft-account path, not a workforce-tenant sign-in. No separate custom OIDC connector is needed for Google, Apple, or Microsoft because External ID federates all three directly; a custom OIDC connector is only relevant for a provider External ID does not natively support, which none of the four require.

**Account linking** (one person, multiple sign-in methods mapping to one user) is explicitly deferred. Each social provider yields a distinct `oid` and therefore a distinct `UserIdentity` row at v1.

## 4. required app registrations and user flows

In the External ID tenant (created by a human in the Azure portal, section 10):

- **API app registration** for `dai-api`. Exposes a scope (recommended `access_as_user`) and an Application ID URI `api://{api-app-id}`. The `dai-api` JWT audience validates to `api://{api-app-id}`. This is the resource the SPA requests a token for.
- **SPA app registration** for `dai-sports-web`. A public client (no secret) of type single-page application, with the redirect and logout URIs in section 9, configured to request the `dai-api` scope.
- **A sign-up/sign-in user flow** (External ID "user flow") that presents the four methods in section 3. The user flow is where Google, Apple, Microsoft, and email are enabled. Federations for Google, Apple, and Microsoft are registered once at the tenant and attached to the user flow.

**Identity-tuple mapping (no schema change), per `next-platform-architecture-plan.md` section 2:** External ID issues every token from its own authority regardless of social provider. Each token carries `oid` (stable per user), `iss` (the External ID tenant issuer), and `idp` (which social provider was used: `google.com`, `apple.com`, `MSA`, or absent for direct email). The auth slice normalizes `idp` to the provider slug (`google.com`->`google`, `apple.com`->`apple`, `MSA`->`microsoft`, absent->`entra`), keeps `Subject = oid`, and sets `Issuer` to the External ID tenant issuer. `IdentityResolver` and `DevProvisionController` both currently hardcode `provider = "entra"` and (for the provision path) read `tid`; External ID tokens have no `tid`, so the auth slice drops `tid`-based tenant splitting and provisions all External ID users into a single platform tenant per deployment at v1.

**JIT provisioning:** for consumer sign-in, the auth slice changes `IdentityResolver` so a valid External ID JWT with no matching `UserIdentity` row triggers just-in-time creation of the `Tenant` (single platform tenant), `User`, and `UserIdentity` rows in one transaction, replacing the manual `DevProvisionController` prerequisite. The dev bypass path is untouched. JIT lands when auth is turned on, not before.

## 5. required environment variables

**`dai-api` (.NET, `Section__Key` convention):** keep the existing `AzureAd` section name (a conventional Microsoft.Identity name even for External ID; renaming churns code and config for no benefit). Add an explicit instance so the authority host is configurable and is no longer hardcoded to `login.microsoftonline.com`.

| env var | purpose | secret? |
|---|---|---|
| `AzureAd__Instance` | External ID host, e.g. `https://{tenant-subdomain}.ciamlogin.com/` (new; replaces the hardcoded workforce host) | no |
| `AzureAd__TenantId` | External ID tenant id (GUID) | no |
| `AzureAd__Audience` | `api://{api-app-id}` from the API app registration | no |
| `Dev__EnableBypassAuth` | `false` in cloud; `true` only in local dev | no |

The auth slice builds the authority as `{Instance}{TenantId}/v2.0` instead of the hardcoded `https://login.microsoftonline.com/{TenantId}/v2.0`. None of these three `AzureAd` values are secrets; they stay plain Container Apps env values, not Key Vault refs (consistent with the provisioning plan section 10). No new secret is introduced by customer auth: External ID public-client SPA sign-in uses no client secret, and the API validates tokens with no secret.

**`dai-sports-web` (Angular):** MSAL configuration (client id, authority, the `dai-api` scope, redirect URIs) is build-time configuration baked into `environment.ts`, not runtime env vars, exactly like `apiBaseUrl` in the provisioning plan section 13. The values are not secrets (SPA public client). They differ per environment (local vs cloud) and are set at build time.

## 6. local dev auth behavior

**Local dev does not change and does not require External ID.** The dev bypass stays the local path:

- `ASPNETCORE_ENVIRONMENT=Development`, `Dev:EnableBypassAuth=true`, `Dev:TenantKey`/`Dev:UserKey` set (as in `appsettings.Development.example.json`). `IdentityResolver` resolves the configured dev user without a token. The sports-app continues to call the API with no bearer token. This is the current, working local workflow and the slice preserves it.
- A developer who wants to exercise real External ID sign-in locally points the Angular MSAL config and the .NET `AzureAd__Instance`/`TenantId`/`Audience` at the External ID tenant and sets `Dev:EnableBypassAuth=false`. Local redirect URIs (`http://localhost:4201`, section 9) must be registered. This is opt-in for auth testing, not the default dev loop.
- The bypass is structurally safe in cloud: `IdentityResolver.cs:40` honors it only when `env.IsDevelopment()` is also true. In `Production` it cannot activate even if the flag were mistakenly set. `Dev__EnableBypassAuth=false` in cloud is defense-in-depth on top of the environment guard.

## 7. cloud auth behavior

- `ASPNETCORE_ENVIRONMENT=Production`, `Dev__EnableBypassAuth=false`. The dev bypass is doubly disabled (environment guard plus flag). Every request must carry a valid External ID bearer token.
- `dai-api` validates tokens against the External ID authority (`AzureAd__Instance` + `AzureAd__TenantId`) and audience (`AzureAd__Audience`).
- The sports-app (served from `swa-dai-sports-web`) uses MSAL to sign the user in against the External ID user flow, acquires a token for the `dai-api` scope, and an HTTP interceptor attaches `Authorization: Bearer {token}` to API calls.
- `[Authorize]` is added to the first gated sports endpoint (`AgentRunsController`) as part of the auth slice. Whether `SportsReferenceController` (reference data) stays anonymous or is gated is a launch decision; gating it is the safer default once the client always carries a token.
- Internal `dai-api -> dai-analyzer` calls need no user token: that hop is on the Container Apps internal network with internal-only ingress (provisioning plan section 7-8). Service-to-service auth on that hop is out of scope for v1, consistent with `next-platform-architecture-plan.md` section 1.

## 8. cors implications

- `dai-api` CORS is currently hardcoded in `Program.cs:18-30` (`spa` policy) to dev origins only (`https://localhost:4200`, `http://localhost:4201`, and 127.0.0.1 variants), applied at `Program.cs:142`. The deployed `swa-dai-sports-web` origin must be added for cloud (already a provisioning-plan section 13 task). The recommended path remains externalizing allowed origins to config (for example `Cors:AllowedOrigins`) so they vary by environment without a rebuild.
- Auth adds one CORS requirement: the policy must continue to allow the `Authorization` header on cross-origin calls. The existing `spa` policy uses `.AllowAnyHeader()`, which already permits `Authorization`; if origins are externalized, preserve `AllowAnyHeader` (or explicitly allow `Authorization`) so the bearer token is not stripped by CORS.
- The MSAL sign-in redirect itself is browser-to-External-ID, not a call to `dai-api`, so it is not subject to `dai-api` CORS. The External ID side is governed by the SPA app registration's registered redirect URIs (section 9), not by `dai-api` CORS.

## 9. callback / redirect / logout url needs

These are registered on the **SPA app registration** in the External ID tenant (not in `dai-api`, not in `dai-analyzer`). MSAL on the sports-app uses them.

| purpose | local dev | cloud |
|---|---|---|
| redirect URI (sign-in) | `http://localhost:4201` | `https://{swa-fqdn}` (the `swa-dai-sports-web` URL, and any custom domain later) |
| post-logout redirect URI | `http://localhost:4201` | `https://{swa-fqdn}` |

Notes:
- Use the SPA platform type in the app registration (authorization code flow with PKCE; no client secret). MSAL Angular defaults to this.
- The redirect URI is an exact-match origin/path the External ID portal validates; the deployed Static Web Apps URL must be registered before cloud sign-in works, and a custom domain is a second registered URI when it is added.
- No redirect/callback URL is needed on `dai-api`: it is an API resource (validates tokens), not an interactive sign-in client. The only inbound to `dai-api` is the bearer-tokened API call.

## 10. what must happen before public exposure

Gating items, all human/portal actions outside this docs slice. The public `dai-api` ingress must not go live until all are true:

1. **External ID tenant provisioned** (Azure portal). Until it exists there is nothing to validate tokens against and no place to configure social federations. This is the prerequisite for every other auth step (`next-platform-architecture-plan.md` decision 1).
2. **Federations and user flow configured:** Google, Apple, Microsoft, and email enabled on a sign-up/sign-in user flow in the External ID tenant.
3. **App registrations created:** the `dai-api` API app registration (scope + audience) and the `dai-sports-web` SPA app registration (redirect/logout URIs from section 9).
4. **Auth slice implemented (section 12):** authority made configurable and pointed at External ID; `idp`-based provider normalization and JIT provisioning in `IdentityResolver`; `[Authorize]` on `AgentRunsController`; MSAL + bearer interceptor in the sports-app; `environment.ts` MSAL config set per environment.
5. **`Dev__EnableBypassAuth=false` and `ASPNETCORE_ENVIRONMENT=Production`** confirmed in the `dai-api` cloud config.
6. **CORS** allows the deployed web origin and preserves the `Authorization` header (section 8).
7. **End-to-end sign-in tested:** at least one real sign-in per method (or as many as are enabled at launch) producing a valid token that `dai-api` accepts and that JIT-provisions a `UserIdentity` row, before the edge is opened to the public.

Until items 1-3 (portal) are done, the auth slice (item 4) has no validation target; it can be written but not verified, so it should land together with, not before, the tenant.

## 11. deployment doc reconciliation

This document is the single source for customer auth readiness. Reconciliation performed:

- `azure-container-apps-provisioning-plan-v1.md` section 14 is updated to point here and to state plainly that the customer path is External ID with a configurable authority instance (it previously flagged the workforce-vs-External-ID authority shape as an open detail; it now names the resolution and references this doc).
- `cloud-deploy-readiness-v1.md` already names "Entra External ID" as the customer-auth target (section 9 and blocker 3) and does not need a correction; it remains accurate. Its `AzureAd__TenantId` "Entra tenant id" wording should be read as the External ID tenant id.
- `next-platform-architecture-plan.md` section 1-2 remains the deeper design narrative (claim mapping, JIT, MSAL interceptor, resolver changes). This readiness doc is the launch-gate summary that sits on top of it; where they could be read to differ, the External ID customer path stated here governs.

## 12. implementation posture: docs-only this slice

Per the slice guardrail (do not implement full auth unless there is a safe, narrow config correction), **this slice implements no auth code.** The one narrow correction available now is making the JWT authority configurable (add `AzureAd:Instance`, build `{Instance}{TenantId}/v2.0`, replacing the hardcoded `login.microsoftonline.com` in `Program.cs:47`). It is deliberately deferred to the auth implementation slice rather than applied here, because:

- changing the JWT authority is only verifiable against a live External ID tenant, which does not exist yet (section 10 item 1 is a gating manual blocker). A config change with no validation target is not a "safe" correction; it is an untested one.
- the authority change belongs with the rest of the auth slice (provider normalization, JIT, `[Authorize]`, MSAL) so the whole path is tested once against the real tenant, not piecemeal.

The correction is low-risk and one line; this doc specifies it precisely so the auth slice can apply and test it in one pass.

## next recommended slice

**Customer Auth Implementation v1**, gated on the External ID tenant existing (section 10 items 1-3, a human portal task). When the tenant is ready, the slice, in order: (a) make the `dai-api` authority configurable via `AzureAd:Instance` and point it at External ID; (b) normalize the `idp` claim to provider slugs and add JIT provisioning in `IdentityResolver`, dropping `tid`-based tenant splitting; (c) add `[Authorize]` to `AgentRunsController`; (d) add MSAL Angular plus a bearer HTTP interceptor to the sports-app and set `environment.ts` MSAL config; (e) externalize CORS origins and preserve the `Authorization` header; (f) test one real sign-in per enabled method end-to-end with `Dev__EnableBypassAuth=false`. This is the last gate before public exposure of `dai-api`.

## future azure / aks / functions / postgres alignment

- **Container Apps:** External ID changes nothing about the topology in the provisioning plan. `dai-api` stays the only public ingress and is the token validator; `dai-analyzer` stays internal-only and needs no user token; `swa-dai-sports-web` is the MSAL public client. The authority and audience are plain env values, not Key Vault secrets.
- **Azure Functions (deferred):** when timer/event functions arrive, they call `dai-api` internal endpoints with a shared-secret header, not a user bearer token (`next-platform-architecture-plan.md` section 7). Customer auth (External ID) governs only the interactive user path, never the platform-job path.
- **Postgres / pgvector (deferred):** unaffected. It is additive calibration/memory reached through the Tool Gateway; it holds no identity rows. `UserIdentity` stays in SQL Server.
- **AKS (deferred):** no auth change on a future AKS move. JWT Bearer validation is host-agnostic; the same authority/audience config travels with the .NET service, exactly as the provisioning plan notes for the container-to-AKS path.
- **Tier enforcement / Stripe (deferred):** identity (who the user is, via External ID) is separate from entitlement (what tier they have, via Stripe). External ID authenticates; a future `TierGuard` reading the `Subscription` table authorizes by tier. This slice defines only authentication.

## references

- `02 Platform/architecture/next-platform-architecture-plan.md` (sections 1-2: original External ID strategy, claim mapping, JIT)
- `02 Platform/architecture/azure-container-apps-provisioning-plan-v1.md` (section 14: auth gating; updated to reference this doc)
- `02 Platform/architecture/cloud-deploy-readiness-v1.md` (sections 9, 11: launch-required auth)
- `02 Platform/architecture/security-and-permissions.md` (explicit-grant, per-tenant, observable posture)
- code: `dai/platform/dotnet/DevCore.Api/Program.cs:42-53`, `Identity/IdentityResolver.cs`, `Controllers/Dev/DevProvisionController.cs`, `DevCore.Domain/Chat/UserIdentity.cs`, `dai/apps/sports-app/src/app/app.config.ts`
