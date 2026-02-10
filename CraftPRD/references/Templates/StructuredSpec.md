# Structured Spec Template

Use this template when restructuring a monolithic PRD via the Structure workflow. This defines the target directory structure and the slimmed-down design document format.

---

## Target Directory Structure

```
<project>/
├── docs/
│   └── design.md                    # Slimmed design doc (~500 lines)
│
├── packages/
│   ├── domain/
│   │   ├── types.ts                 # Core entities, branded types, payload schemas
│   │   ├── state-machine.ts         # XState definition or formal transition table
│   │   ├── invariants.ts            # Assertion functions (run inside transactions)
│   │   └── rules.ts                 # Business rules (priority calc, pick order, thresholds)
│   │
│   └── protocol/
│       ├── commands.ts              # Command names + payload zod schemas
│       ├── events.ts                # Event names (client vs internal) + payload schemas
│       ├── errors.ts                # Error codes + retryable/non-retryable classification
│       └── connection.ts            # Connection lifecycle types (ConnectAck, etc.)
│
├── config/
│   └── schema.ts                    # YAML config schema (zod) with defaults
│
└── src/
    ├── agents/
    │   ├── runner.ts                # AgentRunner abstraction (if applicable)
    │   ├── <agent-name>.ts          # Task definitions (prompt + tools + schema)
    │   └── ...
    └── ...
```

**Not every project needs all of these.** Only create directories/files for areas that were detected in the Structure workflow's analysis phase. Empty directories should not be created "just in case."

---

## Slimmed Design Document Template

After restructuring, the design doc should contain ONLY decisions, rationale, and flows:

```markdown
# <Product Name> — Design Document

**Date:** <ISO date>
**Status:** Draft | Structured | Implementation-Ready

## Problem

<Same as one-pager — concise problem statement>

## Solution Overview

<High-level architecture. The "elevator pitch" of the technical approach.>

## Architecture

<Diagram or brief description of how the major components interact.
Reference code packages: "See `packages/domain/` for entity types and state machine.">

## Design Decisions

Decisions are organized by area. Each decision includes rationale and trade-offs considered.

### <Area 1: e.g., Data Storage>

**Decision:** <what was decided>
**Rationale:** <why>
**Alternatives considered:** <what else was on the table>
**Trade-offs:** <what we gave up>

### <Area 2: e.g., Communication Protocol>

**Decision:** <what was decided>
**Rationale:** <why>
...

## Flows

<High-level user/system flows. NOT the detailed state machine (that's in code), but the "story" of how someone uses the product.>

### <Flow 1: e.g., Happy Path>
1. User does X
2. System responds with Y
3. ...

### <Flow 2: e.g., Error Recovery>
1. When X fails...
2. System recovers by...

## Scope and Boundaries

### In Scope (V1)
- <capability>

### Out of Scope
- <explicitly excluded>

### V2 Ideas
- <future consideration>

## Technical References

| Area | Location | Description |
|------|----------|-------------|
| Domain types | `packages/domain/types.ts` | Core entities and payload schemas |
| State machine | `packages/domain/state-machine.ts` | Full transition rules with guards |
| API contract | `packages/protocol/` | Commands, events, errors |
| Config schema | `config/schema.ts` | All tuneable parameters with defaults |
| Invariants | `packages/domain/invariants.ts` | Assertion functions |
| Business rules | `packages/domain/rules.ts` | Priority calc, pick order, thresholds |
```

---

## Usage Notes

- **The design doc explains "why", the code explains "what".** If you deleted the design doc, someone could still build the system correctly from the code specs — but they wouldn't understand why it was designed that way.
- **Keep it under 500 lines.** If the design doc is growing past 500 lines after restructuring, more content needs to move to code.
- **Technical References table is mandatory.** It's the bridge between prose and code — readers need to know where to find the formal specs.
- **Flows are stories, not specifications.** Use prose to describe what happens, not the exact state transitions (those are in the state machine code).
