---
name: CraftPRD
description: PRD lifecycle from vague idea to structured spec. USE WHEN design product OR write PRD OR review PRD OR structure PRD OR restructure PRD OR one-pager OR "too complex" OR complexity signals OR split PRD.
---

# CraftPRD

Guides the full PRD lifecycle: from vague idea through one-pager to structured technical specification. Detects when a monolithic PRD has outgrown prose and helps split it into the right mediums (code, state machine, prose).

## Core Principle

> **Prose for decisions and rationale. Code for contracts and consistency. State machines for transitions. Don't express technical specs in the wrong medium.**

A PRD naturally grows from a one-pager into a technical spec. At some point (~500-800 lines), it starts containing TypeScript interfaces, SQL schemas, and state machine diagrams in markdown. This is the **complexity threshold** — the signal to restructure into proper mediums where compilers, not humans, enforce consistency.

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **Explore** | "vague idea", "new product idea", "let's think about" | `Workflows/Explore.md` |
| **Draft** | "write it up", "capture this", "one-pager" | `Workflows/Draft.md` |
| **Deepen** | "flesh out", "add detail", "iterate on PRD" | `Workflows/Deepen.md` |
| **Structure** | "structure this", "split PRD", "restructure", "too complex" | `Workflows/Structure.md` |
| **Review** | "review PRD", "find issues", "consistency check" | `Workflows/Review.md` |

## Lifecycle Overview

```
Explore → Draft → Deepen ──→ Structure → Review
 (idea)  (1-pager) (iterate)  ⚠️ threshold  (verify)
                      │                        │
                      └── complexity signals ───┘
                           trigger Structure
```

**Structure workflow adapts to PRD size:**
- Growing PRD (~500 lines): proactive — "split these sections now before it grows"
- Mature PRD (~3000 lines): retroactive — "untangle, migration plan, step-by-step restructure"

## Key References

- [UsualSuspects](references/UsualSuspects.md) — Areas to analyze when destructuring a PRD (domain model, state machine, API contract, framework decisions)
- [ComplexitySignals](references/ComplexitySignals.md) — How to detect when a PRD needs restructuring
- [MediumGuide](references/MediumGuide.md) — What belongs in code vs prose vs state machine vs config

## Examples

**Example 1: Start from scratch**
```
User: "I have an idea for a review pipeline tool"
→ Invokes Explore workflow
→ Structured discussion, captures decisions
→ Output: discussion notes file
```

**Example 2: Write up an idea**
```
User: "Let's write up that review pipeline idea as a one-pager"
→ Invokes Draft workflow
→ Creates one-pager PRD from discussion notes
→ Output: one-pager PRD file
```

**Example 3: PRD getting complex**
```
User: "This PRD has grown to 2000 lines and I keep finding inconsistencies"
→ Invokes Structure workflow
→ Analyzes PRD for complexity signals
→ Walks through "usual suspects" area by area
→ Produces migration plan
→ Executes restructuring step by step
```

**Example 4: Review structured spec**
```
User: "Review this PRD for inconsistencies"
→ Invokes Review workflow
→ Cross-checks types, state machine, API contract
→ Reports findings by severity
```
