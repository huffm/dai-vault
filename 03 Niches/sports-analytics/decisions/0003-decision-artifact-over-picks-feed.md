# decision 0003: decision artifact over picks feed

**date:** 2026-04-27
**status:** accepted
**applies to:** sports-analytics product category, v1 workflow, and user experience

---

## decision

the sports product is positioned as a pre-game **decision artifact** instead of:

- a picks feed
- a live odds board
- a sportsbook execution layer
- a social picks community

the primary unit of value is a structured read for a single game. each read must show the current market context, the reasoning signals, the areas of agreement and conflict, and enough timing information for the user to judge whether the read is still fresh.

---

## why this decision was made

### incumbents already dominate the obvious feature race

the market already contains mature tools for:

- line shopping
- positive EV scanning
- expert picks
- odds alerts
- prop research
- sportsbook routing

matching them on breadth would create a larger product with more noise and a slower path to trust.

### the unmet user need is clarity with receipts

serious recreational bettors do not just want more data. they want:

- faster synthesis
- visible reasoning
- less workflow friction
- proof that prior reads can be judged later

the decision artifact format addresses that need directly.

### this is the right solo-founder fight

the product can win on:

- speed to comprehension
- consistency of format
- trust through transparency
- quality of presentation

those are realistic advantages for a focused product team. a giant execution stack is not.

---

## what the artifact must contain

every v1 artifact should include:

- game header with kickoff or tip-off time
- market snapshot: open line, current line, and update time
- signal summary table
- narrative synthesis
- explicit signal conflict notes when present
- confidence flag with explanation
- watch items or invalidation notes
- archived dispatch snapshot for later review

---

## what was considered and rejected

### picks feed
rejected because it pushes us toward a tout posture and hides the real differentiator, which is reasoning quality.

### live odds board
rejected because incumbents are already strong here and because a board increases surface area before we have earned trust.

### sportsbook execution
rejected because it adds integration and compliance complexity without solving the core user problem first.

### community-first product
rejected because user-generated picks create more noise than clarity in v1.

---

## implications for product design

- the default experience should start from a single game read, not a giant dashboard
- history matters because prior artifacts are part of the trust loop
- visual design should emphasize legibility, not density
- re-alerts should explain what changed, not only that something changed
- every premium feature should strengthen clarity, timing, or accountability

---

## what would need to be true before changing this

- users consistently ask for execution or board features more than they ask for better reads
- the artifact and archive loop no longer feels distinctive
- we can add adjacent features without diluting the core promise of transparent, low-noise decision support
