# scroll reveal and hover lift — design decisions

**Date:** 2026-04-27
**Status:** applied

---

## decisions

### 1. scroll reveal is landing-only; inner pages are flat on load

The scroll reveal entrance animation (`ScrollRevealDirective`) is applied only to the landing page. History, account, and analyzer load their content immediately with no entrance animation.

**Why:** scroll reveal is a sequential discovery pattern suited to marketing/overview pages where you want the user to read through content in order. Inner product pages are navigated to with intent — the user knows what's there. Staggered card-by-card entrance on a repeat-visit page like History adds latency without adding value, and a settings-style page like Account feels theatrical with cascading section reveals.

**Rule:** do not apply `appScrollReveal` to inner app pages. If route transition polish is needed later, handle it at the router level (single fade, not per-element stagger) so it applies consistently and can be tuned in one place.

### 2. scroll-reveal CSS lives in global styles

The `.scroll-reveal` and `.scroll-reveal--visible` class rules are defined in `src/styles.scss`, not in any component SCSS file. The directive is moved to `src/app/scroll-reveal.directive.ts` (shared root location).

**Why:** Angular's view encapsulation scopes component SCSS to the host component's `_ngcontent` attribute. A directive that adds `.scroll-reveal` via `Renderer2.addClass` would be invisible to the scoped CSS of any component other than the one that defines it. Global styles are the only correct home for classes applied dynamically by a cross-component directive.

### 3. hover lift on landing hero content card

The left hero card (`.landing-hero__content`) now lifts `-2px` on hover, matching every other interactive surface card on the page (preview card, feature cards, coverage cards, trust panel metrics, final CTA).

**Rule:** all top-level surface cards receive the standard hover lift (`translateY(-2px)` to `-3px` depending on card weight). The only exceptions are inline text elements, signal rows, and structural layout wrappers that are not perceived as discrete cards.

**Why:** the left hero card was the only card-shaped surface without hover feedback, which felt like a bug rather than an intentional choice. Consistency across interactive surfaces is the default; any card that breaks from this pattern should have a documented reason.

### 4. reduced-motion handling

The global `@media (prefers-reduced-motion: reduce)` block in `styles.scss` disables scroll-reveal transitions and transforms. Component-level reduced-motion blocks handle hover transitions for landing-specific elements. No scroll-reveal exceptions are needed per-component because the global rule covers all instances of the class.

---

## directive location

`src/app/scroll-reveal.directive.ts` — shared root
Previously lived at `src/app/landing/scroll-reveal.directive.ts` (now orphaned dead code; safe to remove in a cleanup pass).
