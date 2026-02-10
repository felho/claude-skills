# Complexity Signals — When a PRD Needs Restructuring

## Quantitative Signals

| Signal | Threshold | What it means |
|--------|-----------|---------------|
| **File length** | > 800 lines | Too much for one document — humans lose context, LLMs lose consistency |
| **TypeScript code blocks** | > 5 interfaces in markdown | Specs that should be in `.ts` files where compiler enforces consistency |
| **SQL/schema blocks** | > 2 CREATE TABLE in markdown | Data model complex enough to warrant its own module |
| **Cross-references** | > 10 "see section X" links | Concepts are entangled — changes in one section ripple to many others |
| **Repeated enums** | Same enum values in > 2 places | Single source of truth violation — will drift |
| **ASCII diagrams** | State machine + sequence diagram | Formal enough to warrant code (XState, mermaid, or formal table) |

## Qualitative Signals

- **Review findings are about consistency, not content.** When reviews keep finding "field X is defined differently here vs there" rather than "this design decision is wrong," the medium is the problem, not the design.

- **The same concept is described in 3+ places.** E.g., a status enum appears in: data model section, API section, event payload section, state machine section. Each description adds nuance but risks drift.

- **Prose is encoding behavior.** "When status is X AND field Y is null AND the previous status was Z, then..." — this is a guard condition masquerading as a paragraph. It belongs in a state machine or assertion function.

- **Recovery logic creates hidden transitions.** Watchdog/timeout sections describe state transitions that aren't in the main state machine diagram. The diagram is incomplete, but nobody notices because the recovery logic is in a different section.

- **Two transports, same operations.** If the PRD describes both REST and WS for the same commands, with different auth and serialization but same business logic — the protocol layer is complex enough to be its own concern.

- **"V1 design note" / "V1 accepted risk" frequency.** More than 5-10 of these suggests the design is sophisticated enough that trade-offs need formal tracking, not inline notes.

## What to Do When Signals Fire

The Deepen workflow checks these signals after each iteration. When multiple signals fire:

1. **Don't panic** — the PRD isn't broken, it's just outgrown its medium
2. **Don't stop writing** — finish the current thought, then restructure
3. **Run the Structure workflow** — it will walk through the "usual suspects" and create a migration plan
4. **The prose design doc survives** — it just shrinks to decisions, rationale, and flows. The technical contracts move to code.
