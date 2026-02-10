# Medium Guide — What Goes Where

## The Four Mediums

| Medium | Best for | Enforces | Example |
|--------|----------|----------|---------|
| **Prose (markdown)** | Decisions, rationale, flows, trade-offs, "why" | Nothing — human discipline only | "We chose SQLite over Postgres because..." |
| **Code (TypeScript)** | Types, schemas, contracts, validations, invariants | Compiler + runtime (zod) | `interface WorkItem { status: ItemStatus; ... }` |
| **State machine (XState/table)** | Status transitions, guards, side effects | Formal verification, visualization | `raw → enriching (guard: canStartEnrich)` |
| **Config (YAML/JSON)** | Tuneable parameters, feature flags, lists | Schema validation | `queue.max_size: 10` |

## Decision Matrix

Ask these questions for each PRD section:

```
Is it a "why" explanation?
  → YES → Prose (design doc)

Does it define a data shape (types, schemas, payloads)?
  → YES → Code (TypeScript + zod)

Does it describe "when X, go to Y, doing Z"?
  → YES → State machine

Is it a tuneable value (threshold, timeout, list)?
  → YES → Config (YAML)

Does it describe transport/protocol behavior?
  → First decide: use existing framework or custom?
  → If framework: the PRD section shrinks to "we use X"
  → If custom: formalize in code (protocol package)

Does it describe agent behavior (prompt, tools, lifecycle)?
  → Prompt + tools = domain code (agent task definitions)
  → Lifecycle (abort, timeout, retry) = AgentRunner abstraction
```

## What Stays in the Design Doc

After restructuring, the design doc should contain ONLY:

1. **Problem statement** — what are we solving, why
2. **Solution overview** — high-level architecture, key insight
3. **Design decisions + rationale** — the "why" behind each choice, with trade-offs
4. **Flow descriptions** — how the user/system moves through the product (high-level)
5. **What's NOT in scope** — explicit boundaries
6. **V2 ideas** — future considerations
7. **References to code specs** — "See `domain/state-machine.ts` for full transition rules"

**Rule of thumb:** If you deleted the design doc, could someone still implement the system from the code specs alone? They should be able to build it correctly — but they wouldn't understand WHY it was designed that way. The design doc provides the "why."

## Anti-Patterns

| Anti-pattern | Why it's bad | Fix |
|-------------|-------------|-----|
| TypeScript interface in markdown | Compiler can't check it, drifts from implementation | Move to `.ts` file, reference from doc |
| State transitions in prose | Incomplete coverage, scattered preconditions | XState or formal transition table |
| Same enum in 3 places | Will drift — which one is the source of truth? | One definition in code, import everywhere |
| Error codes as a markdown table | No runtime validation, no retryable classification | Enum in code with classification |
| "See section X" more than 10x | Concepts too entangled for one document | Split into focused modules |
| Recovery logic separate from state machine | Creates hidden transitions not in the diagram | Timeout/failure = normal state machine events |
| Custom protocol where framework exists | 500+ lines of spec for solved problems | Socket.IO, tRPC, etc. |
| Per-agent lifecycle boilerplate | Same abort/timeout/retry pattern repeated | AgentRunner abstraction |

## Target Directory Structure (after restructuring)

```
project/
├── docs/
│   └── design.md              # ~500 lines: decisions, rationale, flows
│
├── packages/
│   ├── domain/
│   │   ├── types.ts           # Core entities (WorkItem, Finding, etc.)
│   │   ├── state-machine.ts   # XState definition or transition table
│   │   ├── invariants.ts      # Assertion functions
│   │   └── rules.ts           # Business rules (priority calc, pick order, etc.)
│   │
│   └── protocol/
│       ├── commands.ts        # Command names, payloads (zod schemas)
│       ├── events.ts          # Event names, payloads (zod schemas)
│       ├── errors.ts          # Error codes, retryable classification
│       └── connection.ts      # Connection lifecycle types
│
├── config/
│   └── schema.ts              # YAML config schema (zod)
│
└── src/
    ├── agents/
    │   ├── runner.ts          # AgentRunner abstraction
    │   ├── producer.ts        # Task definition (prompt + tools + schema)
    │   ├── enricher.ts        # Task definition
    │   └── apply.ts           # Task definition
    └── ...
```

This structure ensures: shared types between server and client, compiler-enforced contracts, testable state machine, and a design doc that focuses on what it does best — explaining decisions.
