# Angular implementation plan: sports-v1

**date:** 2026-04-10
**app:** `apps/sports-app/` in the dai monorepo
**current state:** single root component, no routing, no auth, signals-based state with computed-driven UI. `useStubApi: true` in production. the dev surface has grown significantly since this plan was written — see `02 Platform/architecture/current-sports-analysis-flow.md` for the current Angular implementation reality.
**scope:** NFL, NBA, and MLB pre-game brief UI
**note:** this document describes the future product UI (game list, brief view, routing). the current working surface is the matchup analyzer dev app. milestones below describe the path from dev tool to full product.

---

## starting point

the current `apps/sports-app/` is a proof-of-concept shell: one component with a form and a result area, no routing, no feature structure. it calls `POST /api/agent-runs` via `SportsApiService` and renders `summary`, `confidence`, and `factors[]` using Angular signals.

the plan below evolves this app — it does not replace it from scratch. existing signal-based state and `SportsApiService` are the foundation to build on.

---

## route structure

```
/                     → redirect to /games
/games                → game list (lazy-loaded feature)
/games/:gameId        → brief view (co-located with games feature, same lazy chunk)
/account              → account and delivery settings (lazy-loaded feature)
/login                → future auth entry point (stub route for now)
```

### why this shape

- `/games` is the entry point — the only surface free-tier users need to reach immediately
- `/games/:gameId` is a child of the games context, not a separate feature; the brief is always reached from the game list
- `/account` is loaded lazily and rarely visited — no reason to include it in the initial bundle
- `/login` is stubbed now; when auth is added, it becomes a real route without restructuring anything

### desktop layout: secondary outlet for brief panel

on desktop, the brief should appear as a side panel alongside the game list (email-client pattern). this is achieved with a named router outlet inside the games layout component rather than full-page navigation. on mobile, the same `/games/:gameId` route is a full-screen view.

```
games layout component
  ├── <router-outlet>             ← game list always renders here
  └── <router-outlet name="panel"> ← brief appears here on desktop only
```

the games layout component reads a breakpoint signal to decide whether to render the named outlet or allow the route to navigate full-screen. the route config uses the `outlet` property to target the panel on desktop, falling back to primary outlet on mobile.

if the named-outlet pattern adds too much complexity for v1, simplify to full-page navigation for both breakpoints and revisit. the route structure is the same either way — only the outlet targeting changes.

---

## feature boundaries

```
apps/sports-app/
  src/
    app/
      core/
        models/           ← AgentRunResultDto, SportsMatchupInput, GameBrief, SignalRow
        services/         ← SportsApiService, GamesService, AuthService (stub)
        guards/           ← AuthGuard (stub), TierGuard (stub)
      shared/
        components/       ← reusable presentational components (see below)
        pipes/            ← confidence-label.pipe, signal-direction.pipe
      features/
        games/            ← lazy-loaded; owns game list and brief view
          components/
            game-list/
            game-card/
            brief-view/
            signal-table/
            alert-banner/
            upgrade-prompt/
          games.routes.ts
        account/          ← lazy-loaded; owns settings and delivery config
          components/
            account-shell/
            delivery-settings/
          account.routes.ts
      app.routes.ts       ← top-level route config, lazy loads features
      app.config.ts       ← provideRouter, provideHttpClient, etc.
      app.ts              ← root shell component
```

### why separate `core/` from `shared/`

`core/` holds things that are singletons or domain-specific: services, models, guards. `shared/` holds presentational components that have no dependency on services or business logic. this boundary makes it obvious whether a component can be moved or reused without side effects.

---

## reusable components (`shared/components/`)

these have no dependency on feature services. they receive inputs and emit outputs only.

| component | inputs | purpose |
|---|---|---|
| `SportBadgeComponent` | `sport: 'nfl' \| 'nba'` | colored pill badge — NFL / NBA |
| `SignalDirectionComponent` | `direction: 'up' \| 'neutral' \| 'down' \| 'null'` | renders ↑ / — / ↓ / null indicator |
| `ConfidenceLabelComponent` | `level: 'high' \| 'medium' \| 'low'`, `reason: string` | confidence row with supporting sentence |
| `BriefStatusBadgeComponent` | `status: 'ready' \| 'updating' \| 'pending' \| 'locked'` | game-card status indicator |
| `LockOverlayComponent` | `tier: string` | replaces locked content in brief view, emits `upgrade` event |
| `SkeletonBlockComponent` | `lines: number`, `width?: string` | loading placeholder for brief content |

these are the only components that belong in `shared/`. anything that calls a service or knows about domain state belongs in a feature.

---

## feature-specific components (`features/games/components/`)

| component | owns | notes |
|---|---|---|
| `GameListComponent` | game list route | fetches today's games via `GamesService`, renders `GameCardComponent` list |
| `GameCardComponent` | individual card | receives `game` input, emits `selected` output; uses shared badge and status components |
| `BriefViewComponent` | brief route | fetches brief via `SportsApiService` by gameId, renders signal table and narrative |
| `SignalTableComponent` | signal table | receives `signals: SignalRow[]` input; renders the five-row scored table |
| `AlertBannerComponent` | alert state | receives `alert` input; conditionally rendered at top of brief view |
| `UpgradePromptComponent` | locked state | rendered inside brief view when tier gate blocks content; game-aware messaging |

`BriefViewComponent` is intentionally not in `shared/` — it knows about routing context, fetches its own data, and manages its own loading state.

---

## where signals make sense

signals are already used in the app for loading and result state. extend that pattern consistently — do not mix signals and RxJS observables for the same state.

**use computed signals for:**
- derived game list state: `gamesReady = computed(() => this.games().filter(g => g.briefStatus === 'ready'))`
- brief loading state: `isLoading = computed(() => this.status() === 'loading')`
- display-only transforms: formatted confidence, spread display, time-to-game countdown

**use signals (writable) for:**
- `games`: the loaded game list
- `selectedGame`: which card is active (drives desktop panel)
- `brief`: the loaded brief for the current game
- `loadingStatus`: `'idle' | 'loading' | 'error' | 'ready'` — typed union, not a boolean flag

**do not use signals for:**
- HTTP calls — keep those as `HttpClient` observables, convert to signals at the boundary with `toSignal()` or explicit subscription in the component
- route params — use `ActivatedRoute.params` as an observable and convert with `toSignal()` if needed

**avoid effect() for data fetching** — use `toSignal()` with `inject(ActivatedRoute)` to derive the brief-load trigger from the route param. effects for data fetching create ordering and cleanup complexity that `toSignal` avoids.

---

## modern template control flow

use `@if`, `@for`, and `@switch` throughout — not `*ngIf`, `*ngFor`, `*ngSwitch`. the app uses standalone components and Angular 17+ already, so there is no reason to use structural directives.

**game list:**
```
@for (game of games(); track game.gameId) {
  <app-game-card [game]="game" (selected)="onSelect($event)" />
} @empty {
  <p>no games scheduled today</p>
}
```

**brief loading state:**
```
@switch (status()) {
  @case ('loading') { <app-skeleton-block [lines]="5" /> }
  @case ('ready')   { <app-signal-table [signals]="brief()!.signals" /> }
  @case ('error')   { <p>brief unavailable</p> }
}
```

**alert banner (conditional):**
```
@if (brief()?.alert) {
  <app-alert-banner [alert]="brief()!.alert!" />
}
```

---

## responsive design approach

### layout strategy: CSS grid with a breakpoint signal

the games layout component uses a CSS grid that switches between one and two columns at a breakpoint. no JavaScript media query listener needed — use a CSS container query on the layout host.

```css
/* games-layout.component.css */
:host {
  display: grid;
  grid-template-columns: 1fr;
  container-type: inline-size;
}

@container (min-width: 720px) {
  :host {
    grid-template-columns: 360px 1fr;
  }
}
```

the game list always occupies the first column. the brief panel occupies the second column on wide viewports and is full-screen on narrow ones.

### breakpoint signal for conditional outlet behavior

a single `BreakpointService` in `core/services/` exposes an `isWide` signal using `BreakpointObserver` converted with `toSignal()`. the games layout component reads this signal to decide:
- wide: brief route targets the named `panel` outlet
- narrow: brief route navigates full-screen on the primary outlet

this is the only place responsive logic requires JavaScript. everything else is pure CSS.

### brief view layout

the signal table and narrative both have a natural max-width of roughly 640px. on any viewport wider than that, constrain the brief container width rather than stretching the content. do not reflow the signal table into a different layout on mobile — it is already compact enough as a five-row table.

---

## GamesService

the app currently has only `SportsApiService` (which submits a matchup form). a `GamesService` is needed to power the game list.

`GamesService` in `core/services/` is responsible for:
- fetching today's scheduled games and their brief statuses from the API
- polling for brief-ready status updates at a low frequency (every 60 seconds is enough)
- providing a `games` signal that the game list component reads

in v1, the games endpoint may not exist yet — the service can be backed by a stub initially and wired to a real endpoint when it is available. the component does not change either way.

`SportsApiService` retains its current responsibility: submitting a matchup request and returning an `AgentRunResultDto`. in the longer term, the brief view loads a brief by `gameId` rather than submitting a form, but that transition can happen without restructuring the component tree.

---

## incremental milestones

### milestone 1 — route structure and feature shells
- add routing to `app.routes.ts`: `/games`, `/games/:gameId`, `/account`
- create lazy-loaded `games` and `account` feature modules with empty shell components
- replace the current single-component root with a minimal shell that redirects to `/games`
- game list renders static placeholder cards

### milestone 2 — game list with real or stub data
- implement `GamesService` with a stub data source (hardcoded game objects)
- implement `GameListComponent` and `GameCardComponent` using signals and `@for`
- implement `SportBadgeComponent` and `BriefStatusBadgeComponent` in `shared/`
- game list is scrollable, cards are tappable

### milestone 3 — brief view connected to current API
- implement `BriefViewComponent` that calls `SportsApiService` by gameId (or matchup input initially)
- implement `SignalTableComponent` with hardcoded signal row data (factors from `AgentRunResultDto`)
- wire the brief route — tapping a card navigates to `/games/:gameId`
- brief view renders summary and factors from the real `AgentRunResultDto`

### milestone 4 — responsive layout
- implement `GamesLayoutComponent` with CSS grid and container query
- add `BreakpointService` for named-outlet routing on desktop
- brief opens as a panel on desktop, full-screen on mobile
- `SkeletonBlockComponent` shows during brief load

### milestone 5 — signal table from real scored output
- this milestone depends on FastAPI returning structured signal data (not just freeform `factors[]`)
- `SignalTableComponent` renders real scored rows with `SignalDirectionComponent` per row
- `ConfidenceLabelComponent` renders the confidence level and reason
- `AlertBannerComponent` renders when a brief has an alert attached

### milestone 6 — tier gating and upgrade prompt
- `TierGuard` blocks access to locked briefs based on subscription status
- `UpgradePromptComponent` renders inside `BriefViewComponent` when content is locked
- `LockOverlayComponent` on game cards that are behind the gate

---

## what to defer

| item | reason |
|---|---|
| auth / MSAL integration in sports-app | the current `AgentRunsController` has no `[Authorize]` attribute; auth in the app is premature |
| real `GamesService` API endpoint | the backend does not yet have a games list endpoint; use a stub until it exists |
| named router outlet (desktop panel) | valid to start with full-page brief navigation and add the panel later without restructuring routes |
| signal table structured data | depends on FastAPI returning scored signals, not freeform factors; use freeform rendering until milestone 5 |
| account / settings feature | no delivery preferences exist to configure yet; scaffold the route, leave the component empty |
| onboarding or empty-state animations | not needed until there are real users to impress |
| service worker / PWA | adds build complexity; defer until mobile usage patterns are understood |
