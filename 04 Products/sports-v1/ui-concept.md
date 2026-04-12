# ui concept: sports-v1

**date:** 2026-04-11 (updated)
**product:** NFL, NBA, and MLB pre-game brief
**target:** recreational bettors who research their own picks and want signal context, not a conclusion

---

## current dev app

the matchup analyzer at `apps/sports-app/` (port 4201) is the working dev surface for this product. it is not the full product concept described below — it is the thin input/output tool used during development.

**current design direction:** layered light theme. app shell `#E6ECF3` (cool blue-gray), white card surfaces with a two-layer diffuse shadow (no border), deep navy primary button and anchor color (`#1E3A5F`, hover `#16314F`), dark slate typography (`#0F172A` headings, `#334155` body), blue-gray focus accent (`#5B84B1`), blue-gray factor chips (`#EAF1F8` bg / `#274060` text). not a dark theme. not bright white. result panel has thin dividers between summary, stat tiles, and factors.

**reactive polish in place:** the form disables during analysis submission and re-enables on completion or error (`emitEvent: false` is used to prevent the disable/enable cycle from triggering downstream reactive subscriptions). the analyze button shows an inline spinner while in flight. the result panel shows an animated skeleton during flight. team dropdowns show "loading teams..." while fetching. the game date dropdown shows "loading dates..." or "no upcoming matchups found" depending on state.

**utility area below the Analyze button — dual-mode:** the space below the Analyze button switches between two states using Angular's `@if / @else` control flow, driven by the `analysisDetails` computed signal.

- **before analyze** (pre-submit, analyzing, error): a lightweight two-line strip. line 1 is the `narration` computed — phase-aware status covering idle guidance, in-flight cycling messages, and error state. line 2 is the `contextLine` computed — subordinate context shown only when useful: date source label when both teams are set but no date is picked (`Date source: live schedule` or `Date source: synthetic (dev)`), compact matchup preview when all fields are complete or analysis is in flight. line 2 renders only when non-null; no placeholder space.

- **after analyze** (result present, no error): a compact **Analysis Details** block replaces the strip. four label/value rows: Analysis mode (`Pre-game read`), Team source (`{Sport} team list`), Date mode (`Live schedule` or `Synthetic (dev)`), Signal quality (`Strong` / `Mixed` / `Weak`). same visual language as the result panel section labels — `text-xs`, muted label, medium-weight value, no card or border. on any upstream field change that clears the result, the strip returns automatically.

- **in-flight messages** are advanced by an `interval` observable writing into an `analysisPhase` signal; this is the designed seam for future backend progress events — replace `interval()` with a backend event stream observable, map payloads to phase indices, and `narration` computed works unchanged.

copy throughout is product-friendly and high-level: no raw run types, no agent internals, no chain of thought, no prompt content. error states show a fixed string; backend error detail is not forwarded to the UI.

**result panel diagnostics:** a separate `resultDiagnostics` computed below the confidence tile interprets the confidence band as a plain-language sentence (`Clean signal. The factors lined up.` / `Mixed picture. Some signals lined up, others pulled against each other.` / `Weak signal. Thin data to work with.`). complements the one-word Signal quality label in the utility area without duplicating it.

**dependent field flow:** sport → home team → away team → available game dates. each upstream change cascades: sport change clears teams, dates, gameDate, and result; team change clears dates, gameDate, and result; date change clears result. dates are fetched from the backend only when both teams are set.

**game date stub mode:** in dev (`useStubSchedule: true` in `environment.development.ts`), `SportsApiService.getMatchupDates()` returns a synthetic 14-day window locally without calling the backend. the narration strip shows `Ready to analyze. Using synthetic dates in dev mode.` when this mode is active — no separate label is rendered. in production (`useStubSchedule: false`), the real endpoint is called and returns `[]` until a schedule data source is connected — no synthetic dates are ever returned by the backend.

**post-analyze state:** after analysis completes, all form selections stay populated. the result panel shows a compact "Analyzed Matchup" section at the top with sport, matchup (away @ home), and date — so the user knows exactly what the result corresponds to. this section clears alongside the result if any upstream field changes.

**team selection:** sport and team inputs are pre-populated dropdowns backed by SQL reference data — not free text. sport has no default — user must select explicitly on each session. each team dropdown excludes the value selected in the other to prevent same-team matchups.

**Angular patterns in use:** signals and `computed()` for reactive state and all derived presentation; `toSignal()` to bridge `valueChanges` observables into the signal graph; `@if` / `@for` control flow blocks. observables remain the path for async and event sources — they are bridged into signals at the rendering boundary, not replaced by signals. latest Angular features are used where they add real value; no unnecessary abstraction is introduced.

this design direction should carry forward into the product UI. the game list and brief view described below should follow the same palette.

---

## primary user jobs

1. **check today's games** — see which games have briefs ready before they need to act
2. **read a brief** — scan the signal table, read the narrative, form a view in under 90 seconds
3. **catch an update** — know immediately when a line has moved after a brief was delivered
4. **decide whether to upgrade** — understand what paid tiers unlock without a sales pitch

these are the only jobs the UI needs to support in v1. everything else is deferred.

---

## key screens

### 1. game list

the entry point. one card per game, ordered by time to kickoff or tip-off (soonest first). this is not a dashboard — it is a feed with a purpose.

each card shows:
- sport badge (NFL / NBA / MLB) and matchup (Away @ Home)
- current spread and total
- time until kickoff or tip-off
- brief status: `ready` / `updating` (line movement alert in progress) / `no brief` (free tier gate)
- aggregate signal indicator — a single directional marker (lean one way, lean other way, mixed) — not a score, not a pick
- lock icon if the game is behind a tier gate

what the list does not show: narratives, signal breakdowns, confidence floats. those live inside the brief. the list is for orientation, not analysis.

**mobile:** single column of cards. sport badge and lock icon are the first things that draw the eye. time to game is prominent. no horizontal scrolling.

**desktop:** same single column, max width constrained to roughly reading width (680–720px), centered. no multi-column grid — this is not a stats dashboard.

---

### 2. brief view

the primary content surface. opens from a game card. full-screen on mobile, modal or focused panel on desktop.

layout, top to bottom:

**header** — matchup, current spread and total, kickoff or tip-off time. one line. always visible.

**signal table** — five rows, one per signal category. each row: label, directional indicator (↑ favorable / — neutral / ↓ unfavorable), and a single short flag phrase (e.g. "line moved 2.5 pts against 68% public"). no raw scores, no percentages in the table itself. null signals (weather for NBA, missing data) shown as a dash with a muted style — not hidden.

**narrative** — 3–5 sentences below the table. plain prose. no bullets, no headers inside the narrative. this is the synthesis, not a list of facts.

**confidence label** — `high` / `medium` / `low` with one supporting sentence. sits between the table and the narrative, or just below the narrative — either works. should not dominate visually.

**alert banner** — only present on threshold-alert briefs. appears at the top of the brief, above the header. one sentence: what moved, by how much, when. labeled clearly as an update.

**mobile:** full-screen takeover. signal table is the first thing visible without scrolling (or nearly so). narrative is below the fold but reachable with one scroll. no sticky headers inside the brief.

**desktop:** modal or right-panel slide-in. signal table and narrative visible together without scrolling on most screen sizes.

---

### 3. upgrade prompt

shown in place of locked content (not as a modal interruption). when a free-tier user taps a locked game card, the brief view opens but the signal table and narrative are replaced with a short statement of what they're missing and what tier unlocks it.

**not a feature list** — one sentence about the specific game they tried to open, then a single call to action. example: "NBA briefs are included in starter and pro. see all games for $29/month."

no pricing table, no comparison grid in v1. those belong on a marketing page, not inside the product.

---

### 4. settings / account (minimal)

accessible from a persistent but unobtrusive control (avatar or icon, top right). contains:
- current plan and renewal date
- delivery preferences (email address, webhook URL for pro)
- sign out

nothing else in v1. no notification preferences, no sport toggles, no theme switcher.

---

## information hierarchy

most important → least important, in order:

1. which games have briefs ready right now
2. what the signal picture says about a specific game (signal table)
3. the narrative synthesis
4. the confidence level
5. account and tier status

the UI should reflect this order. games come before briefs. the signal table comes before the narrative. the confidence label is context, not the headline.

the aggregate signal indicator on the game card is the one piece of analysis visible without opening a brief. it should be a directional cue — not a score. users who want the reasoning open the brief. users who just want a scan get the cue.

---

## what should feel instant

- **game list load** — the list of today's games and their brief statuses should render immediately. briefs that are not ready yet show a clear status, not a spinner.
- **brief open** — tapping a game card should open the brief view immediately if the brief is cached. no loading state for a brief that already exists.
- **alert banner** — a line movement update should appear at the top of the brief without requiring a refresh. brief content below the alert does not reload.

---

## what should be deferred

| feature | why |
|---|---|
| search and filtering | the game list is short enough in v1 that filtering adds no value — NFL has at most 16 games on a Sunday, NBA has at most 13 per night |
| historical brief archive | useful once trust is established; not needed to convert a first user |
| signal trend charts | visualizing line movement over time adds build time and clutters the first-use experience |
| notification settings UI | delivery is configured at account creation in v1; no in-app preference center needed |
| dark/light mode toggle | the app uses a light theme with dark anchors — no toggle in v1 |
| sport filter tabs | NFL and NBA have different schedules — the list is inherently sparse enough that mixing them creates no confusion |
| multi-game comparison view | not aligned with the user job; they evaluate one game at a time |
| onboarding flow or tutorial | the brief format is self-explanatory; if it requires explanation, fix the brief |

---

## design principles for implementation

**one decision, one screen** — the brief view is for one game. there is no "next game" navigation inside it. the user returns to the list.

**no empty states that feel like failure** — if no briefs are ready yet, show the games with a clear "brief available at [time]" instead of an empty list.

**the signal table is the product** — it should be the most legible element on the screen. readable at a glance. not nested in a card, not behind a toggle, not below the fold on mobile.

**lock states inform, they don't block** — a locked game card is still navigable. the upgrade prompt shows inside the brief view. the user is never hard-stopped at the list.

**confidence is supporting context** — `low` confidence should not make the brief feel broken. it means signals were mixed or data was incomplete. the UI should reflect that matter-of-factly, not with warning colors or exclamation points.
