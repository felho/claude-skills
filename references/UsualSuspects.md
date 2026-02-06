# Usual Suspects — Areas to Analyze When Structuring a PRD

When a PRD grows beyond the prose threshold, the Structure workflow walks through these areas. For each area, two questions:

1. **Core or commodity?** — Is this our product's unique logic, or can existing tools handle it?
2. **What medium?** — Should this be expressed as code (compiler enforces), prose (human understanding), state machine (formal verification), or config?

## 1. Core Domain Model

**What to look for:** Entity definitions, type hierarchies, relationships between concepts, branded types, payload schemas.

**Typical signal in PRD:** TypeScript interfaces in markdown code blocks, JSON schema examples, "the payload contains..." descriptions scattered across multiple sections.

**Target medium:** TypeScript types in a shared package (`domain/types.ts`). These become the single source of truth — both server and client import from here.

**Key question:** Are the same types defined in multiple places (API section, data model section, event payloads)? If yes, they MUST be extracted to one definition.

## 2. State Machine

**What to look for:** Status enums, transition rules ("from X, if Y happens, go to Z"), guards/preconditions, side effects on transitions, ASCII state diagrams.

**Typical signal in PRD:** An ASCII diagram in one section, prose transition rules in another, preconditions scattered in the API section, watchdog recovery logic in yet another section. Same transitions described 3-4 times in different words.

**Target medium:** Formal state machine definition. Options:
- **XState** — battle-tested library, has visualization tools, type-safe, testable
- **Formal transition table** — simpler, no dependency, still verifiable
- **Robot** — lightweight alternative to XState

**Key question:** Can you draw the complete state diagram from ANY SINGLE section? If not, it's scattered and needs consolidation.

**What this eliminates:** ASCII diagrams that drift from prose, preconditions that don't match the diagram, recovery logic that creates undocumented transitions.

## 3. API Contract (Commands + Events + Payloads)

**What to look for:** REST endpoints, WebSocket commands, event types, request/response schemas, error codes, ACK payloads.

**Typical signal in PRD:** TypeScript interfaces for payloads in markdown, discriminated unions for event types, error code enums, "the server responds with..." scattered across command descriptions.

**Target medium:** Shared TypeScript package with zod schemas (`protocol/commands.ts`, `protocol/events.ts`, `protocol/errors.ts`). Runtime validation + type safety in one place.

**Key question:** If a frontend developer reads ONLY the protocol package (no prose), can they build a correct client? If not, the contract is incomplete.

## 4. Communication / Realtime Protocol

**What to look for:** WebSocket connection management, message ordering, reconnection strategy, gap recovery, idempotency, auth on transport level.

**Typical signal in PRD:** Custom protocol design (correlationId, serverSeq, sessionToken, connection binding). 500+ lines describing transport-level concerns.

**Decision point — Core or commodity?**
- If the protocol IS the product (e.g., building a messaging platform): custom, but formalize as a separate protocol spec.
- If the protocol is infrastructure (e.g., app needs realtime updates): **use existing tools.**

**Existing tools to consider:**
- **Socket.IO** — reconnection, ack, rooms built-in. Best for bidirectional + streaming.
- **tRPC subscriptions** — type-safe RPC + server→client events. Best for typed API + events.
- **tRPC + Socket.IO hybrid** — tRPC for commands/queries (REST), Socket.IO for realtime only.
- **GraphQL subscriptions** — typed schema, good ecosystem, heavier setup.

**What this eliminates:** Custom serverSeq ordering, gap recovery logic, correlationId matching, reconnection state management, idempotency log for transport-level dedup.

## 5. Agent / LLM Interaction

**What to look for:** Multiple agent types with similar lifecycle (prompt building, tool setup, SDK calls, abort handling, timeout, retry, cleanup).

**Typical signal in PRD:** Each agent described in its own section with repeated boilerplate (AbortController, processingAttempt, watchdog recovery, session lifecycle). "No zombie generations" concerns repeated per agent type.

**Decision point — Core or commodity?**
- The prompt templates and tool definitions ARE core (they define what your agents do).
- The lifecycle management (abort, timeout, retry, guard) is NOT core — it's the same for every agent.

**Recommended pattern:** Thin `AgentRunner` abstraction:
```typescript
interface AgentTask<TInput, TOutput> {
  name: string;
  mode: "headless" | "interactive";
  buildPrompt: (input: TInput) => string;
  tools: ToolDefinition[];
  outputSchema: ZodSchema<TOutput>;
  timeout: number;
  maxAttempts: number;
  toolCallGuard?: (toolName: string, input: unknown) => boolean;
}
```

**What this eliminates:** Repeated abort/timeout/retry/cleanup logic per agent. "Enricher hang recovery" and "Apply crash recovery" become `timeout + maxAttempts` in the task definition.

## 6. Invariants

**What to look for:** "X must always be true", assertion functions, post-transaction guards, consistency checks.

**Typical signal in PRD:** Invariant tables in prose, assertion function code in markdown, "if this invariant is violated, it's a bug" statements.

**Target medium:** Code — assertion functions (`domain/invariants.ts`). Run inside transactions, tested explicitly.

**Key question:** Are all invariants checkable from a single function call? If they're scattered across prose, some will be missed during implementation.

## 7. Error Handling

**What to look for:** Error code enums, retryable vs non-retryable classification, error-to-HTTP-status mapping, "on error X, do Y" rules.

**Target medium:** Code — enum + classification (`protocol/errors.ts`). The retryable/non-retryable distinction is especially important to formalize.

## 8. Business Rules

**What to look for:** Calculations (priority, scoring), ordering logic (queue pick order), threshold rules (backpressure), scheduling (round-robin).

**Target medium:** Code for the logic + prose for the rationale. The "what" is code (`domain/rules.ts`), the "why" stays in the design doc.

**Key question:** Can you test the business rule without reading the prose? If not, the rule isn't sufficiently formalized.

## 9. Security Model

**What to look for:** Auth tokens, session management, CSRF protection, permission model.

**Decision point:** If simple (single localhost token): a section in the design doc + middleware code. If complex (multi-user, OAuth, ACL): separate security spec.

## 10. Recovery / Resilience

**What to look for:** Timeout values, watchdog logic, crash recovery, lease TTL, cleanup sweeps.

**Typical signal in PRD:** Recovery logic described separately from the normal flow, creating implicit state transitions that aren't in the state machine diagram.

**Target medium:** Mostly absorbed by framework choices (Socket.IO reconnect, AgentRunner timeout) and the state machine (timeout events = normal transitions). What remains is configuration (TTL values → YAML config).

## 11. Configuration

**What to look for:** Tuneable parameters (thresholds, timeouts, concurrency limits), persona lists, file paths.

**Target medium:** YAML schema with defaults. The design doc explains what each parameter does; the schema defines valid values.

## 12. UI Specification

**What to look for:** Wireframes, panel layouts, interaction patterns, UX copy.

**Target medium:** Stays in prose / design tool. UI specs rarely benefit from code formalization (except for shared types which are already in the protocol package).

---

## The Structure Workflow Process

For each area above:

1. **Detect:** "Is this area present in the PRD?" (scan for signals)
2. **Assess:** "How many lines? How many cross-references? Is it scattered?"
3. **Decide:** "Core or commodity? What medium?"
4. **Plan:** "Which PRD sections → which target files?"
5. **Execute:** "Extract, create files, update PRD references"

The output is a migration plan that maps PRD sections to target files, with decisions documented for each area.
