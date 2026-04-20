# decision 0002: three-item nav shell with Saved Reads inside History

**date:** 2026-04-19
**status:** accepted
**applies to:** sports-v1 frontend shell and nav structure

---

## decision

the v1 top nav has exactly three items:

1. Matchup Analyzer
2. History
3. Account

Saved Reads is not a top-level nav item. it is a filter tab inside the History page (`All reads` / `Saved reads`).

---

## why three items, not four

a four-item nav (Matchup Analyzer, History, Saved Reads, Account) was considered. it was rejected because:

- **Saved Reads is a view of History, not a separate workflow.** a user who wants saved reads is navigating their prior reads with a filter applied. making it a top-level destination implies it is a peer of History, which it is not — it is a subset of it.
- **four items adds visual weight without adding clarity.** the header is already information-bearing (brand, nav, responsive hamburger). a fourth item narrows the visual breathing room and raises the cognitive load for a feature that is reached more naturally via History.
- **three items maps cleanly to three user jobs at this scope:** run an analysis, review past analyses, manage account. there is no fourth job that belongs at the top level in v1.

---

## why History and Account as real mock pages, not dead routes

both pages are fully styled in the same design language as the analyzer. they are not placeholders. rationale:

- **the shell needs to feel coherent.** if nav items route to blank or "coming soon" screens, the product feels broken even in dev. mock pages that follow the same card/surface/typography system establish that these are real product siblings, not orphaned future work.
- **the Account structure captures real product decisions.** even as a static mock, the Account page reflects what v1 account management should contain (plan, delivery, security) and what it should not (notification center, sport toggles, theme switcher). this is the correct reference for when the backend is wired.
- **the History page establishes information architecture.** the card structure, sport badges, confidence bands, and grounded signal chips reflect real data the system will eventually surface. designing this now, against real mock data, validates the layout before the api integration work begins.

---

## what this decision does not include

- backend history persistence — not implemented. saved reads are client-side signal state only.
- real account management — not implemented. account page is static.
- route guards or auth-gating — not in scope for v1 shell.
- a game-list entry point replacing the analyzer as the default — the analyzer remains the default route (`/`). a game-list view is a future product screen described in `ui-concept.md`; it does not exist yet.

---

## what would need to be true before changing this

- adding a fourth top-level nav item requires a user job that does not fit inside an existing section and is accessed as a primary workflow, not a filtered sub-view
- moving Saved Reads to the top nav would require evidence that users treat it as a peer workflow to History, not a subset of it
- the three-item structure should remain stable through at least the first wired History and Account implementations
