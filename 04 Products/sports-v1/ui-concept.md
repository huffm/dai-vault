# ui concept: sports-v1

**date:** 2026-04-19 (competition-first selector slice added)
**product:** football, basketball, and baseball pre-game brief with competition-aware routing
**target:** recreational bettors who research their own picks and want signal context, not a conclusion

---

## current dev app

the matchup analyzer at `apps/sports-app/` (port 4201) is the working dev surface for this product. it now has a thin app shell with a persistent header and three routed pages. it is not the full product concept described below — the product concept describes the future brief/game-list surface; the dev app is the thin input/output tool used during development.

the analyzer is now **competition-aware**:
- user-facing flow: `sport` → `level` → `team a` / `team b` → `game date` → `analyze`
- internal routing: explicit competition code
- currently supported combinations:
  - football + pro → NFL
  - football + college → NCAAF
  - basketball + pro → NBA
  - basketball + college → NCAAMB
  - baseball + pro → MLB
- baseball + college is visible as unavailable, not hidden and not faked

---

## app shell (implemented 2026-04-19)

the dev app has a lightweight angular shell with route-based navigation.

**root shell:** `App` component owns the sticky header and a `<router-outlet>`. all page components are lazy-loaded standalone components. the shell persists across all routes; page content swaps below it.

**route structure:**
- `/` → redirect to `/analyzer`
- `/analyzer` → `AnalyzerComponent` — the matchup analyzer page; default route
- `/history` → `HistoryComponent` — prior reads list; client-side mock data only
- `/account` → `AccountComponent` — plan, delivery, security shell; static mock only

**top nav (3 items):**
- `Matchup Analyzer` — always the primary action path
- `History` — prior reads with Saved Reads filter inside the page
- `Account` — plan and preferences

**Saved Reads placement decision:** Saved Reads is not a top-level nav item. it lives inside the History page as a filter tab (`All reads` / `Saved reads`). the top nav stays at three items.

**header:** sticky glass nav bar, 64px mobile / 68px desktop. brand ("Sports Analytics") links to `/analyzer`. nav active state is driven by `routerLinkActive` — no hardcoded boolean. mobile: hamburger opens a slide-down drawer; tapping a link closes it automatically. responsive: full nav visible at `lg` breakpoint; hamburger below it.

**History page (mock):** styled using the same card surface system as the analyzer. shows six realistic sample reads with sport badge, matchup, game date, lean, confidence band, and grounded signal chips. Saved Reads is a filter tab within this page. save/unsave toggle is client-side only — no backend persistence.

**Account page (mock):** styled as a natural sibling of the analyzer. sections: Profile, Plan, Delivery, Security. all data is static placeholder — no backend wiring, no real auth, no real billing integration.

**what is NOT wired in history and account:** there is no backend read history. saved reads are not persisted. account data is not real. both pages exist to establish the UI structure and design language before the backend surfaces are built.

---

## current page architecture (analyzer)

the live analyzer page is a single-page scroll surface with four coordinated layers.

- **hero shell**: product label, `Matchup Analyzer` heading, setup copy, and a metadata chip row that appears after sport and both teams are selected.
- **left control rail**: sticky on desktop, stacked in normal flow on mobile. holds sport/level/team/date selection, the primary action, and the supporting utility block below it.
- **right primary card**: `Matchup Read` is the answer surface. it owns the empty state, loading state, error state, analyzed matchup metadata, current lean slot, summary, status, confidence, and diagnostics.
- **full-width supporting section**: `Factor Breakdown` sits below the main row as a wider reasoning layer so factor text can breathe horizontally.

this pattern should remain stable. future game-list and brief surfaces can reuse the same surface language even if the information architecture changes.

**hierarchy decisions:**

- eyebrow pill labels (`PRIMARY READ`, `CONTRIBUTING FACTORS`) are removed. section titles stand on their own. pill labels added unnecessary UI chrome and felt like internal system language rather than product-facing copy.
- unified divider system: `border-b border-[#22344a] pb-4 lg:pb-5` wraps the heading in all four main section headers — hero block, configure matchup, matchup read, factor breakdown. same class, same values, no variants.
- do not add filler cards just to make the page feel more square. do not move the control rail into a full-width section for visual symmetry. do not hide the answer and reasoning behind true tabs. the correct reading model is still:

- `Configure Matchup` = left control rail
- `Matchup Read` = primary answer card in the right column
- `Factor Breakdown` = wider supporting reasoning section below the row
- single-page scroll = default interaction model on both desktop and mobile
- future modules are allowed only if they add real product value, not layout filler or geometry

**current visual system:** dark navy workspace. near-black base field, layered card surfaces with subtle tonal gradients, restrained cobalt accent. visual language influenced by premium desktop product software — clean, dark, and dimensionally hierarchical.

- page field: fixed atmospheric background owned by `body::before`. single smooth `150deg` diagonal linear gradient from deepest near-black at the corners to slightly warmer dark navy toward the lower-right. `fixed` positioning keeps the field continuous on long pages with no seams.
- background tokens (current `styles.css`):
  - html fallback: `#060e18`
  - body::before base: `#060e18`
  - main diagonal: `linear-gradient(150deg, #04080e 0%, #060e18 40%, #091623 100%)`
  - no radial layers — the previous atmospheric radial was removed because its center at `50% -5%` produced a visible blue glow band at the viewport top edge
- card surface classes (defined in `styles.css`):
  - `.card-surface` — hero, control rail, result card: `linear-gradient(160deg, #142238 0%, #0f1b2e 55%, #0c1829 100%)`
  - `.card-surface-deep` — factor breakdown section: `linear-gradient(160deg, #0f1b2e 0%, #0c1829 55%, #091422 100%)`
- inset layer: `#0c1628` — stat cells, empty-state containers, disabled fields
- raised inner surface: `#122033` — lean card, factor cards, empty-state icon chip
- card ring: `ring-1 ring-[#2a3f5a]/80`
- accent: `#2b74ff` — primary button, active date pills, focus rings, empty-state icon tint
- accent hover: `#3b82ff`
- border subtle: `#22344a` — section dividers, control borders, card rings
- border mid: `#29415c` — hover borders, stronger separators
- text primary: `#eef4fb`
- text secondary: `#9eb0c8` — form labels, body copy
- text muted: `#7688a3` — helper text, metadata, placeholders
- focus ring: `rgba(59,130,255,0.35)`

**card depth hierarchy:** four tiers from outermost to most recessed:
1. page field — near-black gradient, no card chrome
2. `.card-surface` — main cards; subtle lighter top-left, deeper bottom-right gradient; `inset 0 1px 0 rgba(255,255,255,0.06)` top edge highlight; `ring-1 ring-[#2a3f5a]/80` border definition
3. `.card-surface-deep` — factor section; one step darker; `inset 0 1px 0 rgba(255,255,255,0.04)` top edge
4. `#0c1628` inset — stat cells and empty-state containers; deepest layer; `inset 0 2px 4px rgba(0,0,0,0.25)` recessed shadow

system font stack is inherited from `styles.css`. hierarchy is built through weight, spacing, tone, and surface tier rather than mixed font families.

**visual direction is locked:** dark navy direction, card surface gradient system, cobalt accent, and hover interaction family are stable. do not reintroduce any light or warm-toned surfaces. do not add radial glow layers to the page background.

**current interaction language:** subtle, shared, and restrained.

- shared control family uses `premium-interactive`, `premium-control`, `premium-button`, `premium-surface`, and `inset-surface`
- real controls get slight lift, richer shadow, and clear cobalt focus ring (`rgba(59,130,255,0.35)`)
- `.premium-surface` hover: border `#29415c`, bg `#16263c`, subtle drop shadow — factor cards and lean card
- `.inset-surface` hover: border `#29415c`, bg `#111e2e`, slight inset reduction + small drop shadow — stat cells (Status, Confidence); more restrained because these start from the deepest base layer (`#0c1628`)
- informational chips and metadata tags stay non-interactive
- active states fill with cobalt (`#2b74ff`); passive metadata chips stay at `#122033`
- do not add extra glows, over-brighten borders, or make informational tiles feel like primary buttons

**control-system decision:** sport, level, team a, and team b now use the same picker family. this is the correct direction and should remain the default unless a future product surface proves it insufficient.

- sport uses the shared picker in non-searchable mode
- level uses the shared picker in non-searchable mode
- team a and team b use the shared picker in searchable mode
- all four controls share the same shell, border, spacing, selected-chip treatment, clear affordance, and dropdown styling
- do not split sport or level back out into one-off native selects or a different custom control

the implementation class is still named `TeamPickerComponent`, but its input surface is already generic. renaming it is not required in this slice and would be churn without product value.

**display-value rule:** internal values may stay lowercase (`nfl`, `ncaaf`, `nba`, `ncaamb`, `mlb`) for contracts and routing. outward-facing UI must always use display labels from the reference data. if a display label lookup ever fails, the fallback should still avoid showing a raw lowercase code.

current outward-facing selection paths:

- sport picker selected chip
- level picker selected chip
- hero metadata chip row
- analyzed matchup metadata
- analysis details block (`{Competition} team list`)

this rule should remain stable across the later game list and brief views.

**lean module pattern:** the top of `Matchup Read` now reserves a compact answer module for the current lean.

- it sits below `Analyzed Matchup` and above `Summary`
- it is a compact highlight module, not a hero banner
- if a structured lean field exists later, it should render there directly
- if no lean field exists, the module must stay neutral and explicit (`Lean unavailable`)
- do not infer a pick from summary text or factor language

**hero metadata behavior:**

- sport chip appears once sport is selected
- level chip appears once level is selected
- matchup chip appears once both teams are selected
- date chip appears once a specific event/date is selected
- `dev` chip appears only when the schedule source is synthetic

**current state model:** the app already handles the important phase changes cleanly.

- idle guidance before a sport is selected
- level selection after sport selection
- team loading after level selection
- date loading after both teams are selected
- ready state once date is selected
- in-flight analysis with rotating narration messages
- error state with fixed product copy
- idle `Matchup Read` uses an anchored inner empty-state panel rather than a loose placeholder
- completed state resolves into an answer-first flow: analyzed matchup, current lean, summary, status/confidence, diagnostics, then factor tiles below the row
- factor tiles are supporting reasoning cards: stronger title, calmer body copy, two-column on larger screens, one-column on smaller screens

the form disables during submission and re-enables on success or error using `emitEvent: false`. upstream changes clear downstream selections and any stale result content. selected matchup context persists after analyze so the user can verify what the result belongs to.

**scroll and background constraints:**

- the sticky control rail must stay structurally confined to the top row
- lower sections like `Factor Breakdown` must live outside that sticky row so date pills, chips, and picker surfaces do not paint over later content during scroll
- the atmospheric page field must belong to a dedicated page-level layer, not a scroll-sized content wrapper, so long populated pages do not show seams or darker restart bands

**utility area below the primary action:** this area intentionally has two modes.

- before analyze: a two-line status strip driven by `narration` and `contextLine`
- after analyze: a compact `Analysis Details` block driven by `analysisDetails`

this split is correct. it keeps the control rail informative before submission and concise after submission without introducing another card or panel.

**accessibility and resilience:** the shared picker family should keep improving, but only in ways that preserve the current interaction model. current direction:

- keep explicit labels on each control
- keep clear buttons on selected chips
- keep keyboard- and screen-reader-friendly dropdown semantics
- keep placeholder text instructional, not decorative

**visual direction is locked.** the dark navy direction, card surface gradient system, cobalt accent, and hover interaction family are stable. do not reopen the visual direction without a clear product reason. do not reintroduce light or warm-toned surfaces.

**near-term UI refinements (open):**

- validate the picker family in browser across keyboard, mobile, and loading states
- browser-check the fixed atmospheric background across empty, populated, and long-scroll states
- keep outward-facing sport family, level, and competition labels display-safe everywhere as the app grows
- preserve the hero shell / main-row rails / full-width factor breakdown pattern while the backend output gets richer
- add a real structured lean field only when the backend contract actually supports it
- improve backend result quality and structure before inventing more frontend chrome
- consider future modules only if they add real user value to the read, not because the page needs another box
- browser-verify the analysis details block across supported competitions, including run time and team-source labeling

this design direction should carry forward into the product UI. the game list and brief view described below should reuse the same sense of hierarchy and restraint even if their layouts differ.

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

### 4. account

accessible from the top nav (`Account`). in the current dev shell this is a styled mock page. the production surface should include:
- profile summary (email, workspace, member since)
- current plan and renewal date
- delivery preferences (email address, webhook URL for pro)
- line movement alert toggle (pro)
- sign out

no notification center, no sport toggles, no theme switcher in v1. the dev mock page reflects this scope — it is a static shell, not a wired account system.

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
| historical brief archive (backend) | the History page shell exists as a client-side mock. real persistence — storing completed runs and surfacing them in History — is deferred until the brief itself is worth archiving |
| signal trend charts | visualizing line movement over time adds build time and clutters the first-use experience |
| notification settings UI | delivery is configured at account creation in v1; no in-app preference center needed |
| dark/light mode toggle | the app uses a dark theme — no toggle in v1 |
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
