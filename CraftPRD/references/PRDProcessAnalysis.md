# PRD Process Analysis: Why Do We Keep Finding Issues?

> **Context**: Analysis of the Bob V1 PRD review pipeline after 109 fix commits across 14 review rounds in ~38 hours. The PRD uses a hybrid prose+code approach (prose for rationale, TypeScript for contracts). This document captures findings from a systematic commit-by-commit analysis and proposes improvements — particularly relevant as input for a **static validator**.

---

## 1. PRD Structure Overview

The Bob V1 PRD is a hybrid prose+code specification: **5,912 lines across 19 files**.

| Category | Files | Lines | Purpose |
|----------|-------|-------|---------|
| Prose (design + findings) | 2 | 2,669 | Main design doc (2,194 lines) + review findings (475 lines) |
| Domain Model | 4 | 1,309 | types.ts (313), state-machine.ts (540), rules.ts (251), invariants.ts (205) |
| Protocol Layer | 6 | 1,094 | dto.ts (252), commands.ts (126), events.ts (457), envelopes.ts (132), errors.ts (127) |
| Infrastructure | 2 | 313 | config/schema.ts (107), db/schema.ts (206) |
| Agent Contracts | 5 | 527 | runner.ts (186), producer.ts (72), enricher.ts (79), apply.ts (96), discussion.ts (94) |

**Key architectural patterns in the code files:**
- Branded types for compile-time safety (UserId, WorkerId, ItemId)
- Pure business logic (rules.ts) separate from state management (state-machine.ts)
- Invariant assertions run inside DB transactions
- Spec-level contracts (agent files) separate interface from implementation
- Single source of truth principle: types imported everywhere, no duplication
- Event-driven architecture with WebSocket envelopes and category metadata

---

## 2. Review Cycle Metrics

### 2.1 Overall Numbers

| Metric | Value |
|--------|-------|
| Total fix commits | **109** |
| Review rounds | **14** |
| Total elapsed time | ~38.7 hours (1.6 days) |
| Effective work time (excl. nights) | ~16-18 hours |
| Avg round interval (excl. nights) | ~22 minutes |

### 2.2 Round-by-Round Breakdown

| Round | Time (CET) | Fixes | Notes |
|-------|------------|-------|-------|
| R1 | Feb 06, 21:45 - 22:24 | 19 | First review on monolith PRD |
| R2 | Feb 06, 22:38 - 22:49 | 6 | Post-ProcessFindings filter |
| R3 | Feb 06, 23:12 - 23:52 | 12 | Third review, still monolith |
| R4 | Feb 07, 00:07 - 00:28 | 3 | R3 continuation (#16-#18) |
| R5 | Feb 07, 00:53 - 01:29 | 16 | First full review of restructured PRD |
| R6 | Feb 07, 09:48 - 10:18 | 17 | Morning review, highest fix count |
| R7 | Feb 07, 10:29 - 10:33 | 3 | Quick follow-up |
| R8 | Feb 07, 10:49 - 10:51 | 2 | Mini round |
| R9 | Feb 07, 13:05 - 13:55 | 6 | Merged fixes (#3+#4, #5+#6), "I" prefixes |
| R10 | Feb 07, 14:25 - 14:28 | 3 | Short round |
| R11 | Feb 07, 15:01 - 15:04 | 4 | Short round |
| R12 | Feb 07, 15:31 - 16:25 | 5 | Afternoon review |
| R13 | Feb 07, 19:54 - 19:55 | 2 | Evening quick fix |
| R14 | Feb 08, 12:12 - 12:27 | 6 | Latest review |

### 2.3 Trend: Decreasing Fix Counts

```
Early rounds  (R1-R5):  19, 6, 12, 3, 16  → avg 11.2 fixes/round
Later rounds (R6-R14): 17, 3, 2, 6, 3, 4, 5, 2, 6  → avg 5.3 fixes/round
```

The trend is **clearly decreasing**, which signals the PRD quality is improving round over round. However, it has not plateaued to zero — we're still finding 2-6 issues per round.

### 2.4 Restructuring Impact

The PRD was restructured from monolith prose to hybrid prose+code on Feb 06 afternoon:

| Phase | Rounds | Fixes | % of total |
|-------|--------|-------|------------|
| Pre-restructuring (monolith) | R1-R4 | 40 | 37% |
| Post-restructuring (hybrid) | R5-R14 | 69 | 63% |

The restructuring exposed more issues (which is good — better visibility), but it also created new issue categories like inter-file cross-reference inconsistency.

---

## 3. Fix Categorization (All 90 Classified Commits)

### 3.1 Summary

| # | Category | Count | % | Automatable? |
|---|----------|-------|---|--------------|
| 4 | Missing documentation/clarification | **24** | 26.7% | Partially |
| 1 | Missing types/interfaces | **16** | 17.8% | **Yes** |
| 3 | Cross-reference inconsistency | **10** | 11.1% | **Yes** |
| 5 | Missing side effects | **10** | 11.1% | **Yes** |
| 2 | Missing guards/validations | **9** | 10.0% | **Yes** |
| 6 | Architectural/design change | **5** | 5.6% | No |
| 7 | Missing error codes/handling | **5** | 5.6% | **Yes** |
| 9 | Code-prose drift | **5** | 5.6% | Partially |
| 8 | Broken references | **4** | 4.4% | **Yes** |
| 10 | Other | **2** | 2.2% | - |

**Key finding: 94% are specification completeness issues, only 6% are genuine design flaws.**

Approximately **55-60% of all issues are potentially automatable** via static validation (categories 1, 2, 3, 5, 7, 8).

### 3.2 Category Details

#### Category 1: Missing Types/Interfaces (16 commits, 17.8%)

The prose described concepts but the code files never formalized them as TypeScript types.

**Representative commits:**
- `2ba6911` — Protocol had WS commands but never defined ACK payloads. Added typed AckPayloadMap.
- `3c5430d` — IdempotencyKey was `string` everywhere, but `idem_` prefix convention was only in prose. Added branded type.
- `a8435d2` — RuntimeState (runSecret, ports, lifecycle) described in prose, no TypeScript interface.
- `42a9192` — ProcessingResetResponse type for REST-only admin endpoint.
- `e00ea8b` — RestErrorResponse type for consistent REST/WS error format.
- `7000f6b` — chat.chunk and chat.done event definitions missing from events.ts.

**Validator opportunity:** Check that every noun/concept referenced in prose that looks like a type (`XxxYyy`, `FooPayload`, `BarResponse`) has a corresponding TypeScript definition.

#### Category 2: Missing Guards/Validations (9 commits, 10.0%)

State machine transitions or endpoints lacked guard conditions that were implied by the design.

**Representative commits:**
- `027ac87` — ENRICH_START had no processingAttempt guard. Enricher could retry forever.
- `4574661` — DECISION_ACCEPT/REJECT had no expectedVersion guard. Optimistic locking in prose, missing in code.
- `7dfa888` — CLAIM composite guard not formalized (status + lease + enriched checks).
- `df167d3` — max_enrichers concurrency check missing from enricher pseudocode.
- `050c124` — processing/reset precondition logic was wrong.

**Validator opportunity:** For each state machine transition, verify it has a `guard` field. Cross-reference guard names with actual guard function definitions. Check that config values like `MAX_PROCESSING_ATTEMPTS` are referenced in the appropriate transition guards.

#### Category 3: Cross-Reference Inconsistency (10 commits, 11.1%)

Field names, enum values, or type shapes diverging between files or between prose and code.

**Representative commits:**
- `520bc75` — State machine emit fields used ad-hoc strings like `"item.enriched"` but ClientEventName enum had different names.
- `6220367` — Prose examples still used `serverSeq` (pre-Socket.IO) while code had `runSeq`.
- `9df091d` — dto.ts and events.ts used inline `"raw" | "enriched" | ...` unions instead of importing `ItemStatus`.
- `70cfe97` — Schema had `INTEGER` for a field that was actually boolean.
- `d91113a` — transcriptMessages was optional in one place, required in another.
- `21a53e1` — AckPayloadMap said it "mirrors" CommandPayloadMap but actually extends it.

**Validator opportunity:** This is the highest-value automation target.
- Verify all enum/union references use the canonical imported type, not inline literals.
- Verify field names referenced in state machine transitions exist in the relevant DTO/type.
- Verify prose code examples use the same field names as the actual code files.
- Verify type assertions (e.g., "INTEGER" in schema) match TypeScript types.

#### Category 4: Missing Documentation/Clarification (24 commits, 26.7%)

Behavior existed by implication but was never explicitly documented.

**Representative commits:**
- `9bc2144` — applyShouldAbort mechanism never written down (35 lines of documentation added).
- `d19a34c` — "single-actor" vs "single-person" ambiguity could mislead implementation.
- `032d1a8` — Reconnect outbox replay order (FIFO) was implicit but critical for correctness.
- `f6f2e16` — concurrent-safe guarantee for abortAndCleanup never documented.
- `82cb7e7` — RuntimeState vs DB-persisted run state difference not clarified.
- `5129df1` — runner.ts was a spec-level contract, but looked like a TODO.
- `b3017bf` — Multi-tab safe scope (browser tabs via WS only) not documented.

**Validator opportunity:** Limited. Some patterns are detectable:
- Functions/methods referenced without JSDoc → flag for documentation.
- Config values used without describing semantics.
- "TODO" or ambiguous wording detection.
But many of these are genuinely about human intent that a validator cannot infer.

#### Category 5: Missing Side Effects (10 commits, 11.1%)

State machine transitions listed target state but omitted field clears/sets that need to happen.

**Representative commits:**
- `5f6d065` — processingStage missing from sideEffects in ALL 11 transitions.
- `33eee93` — conflictNote not cleared on needs_reenrich paths (stale data would display).
- `3caa91b` — sourceHashStale not cleared on APPLY_SUCCESS (unnecessary re-validation).
- `0973262` — applyShouldAbort not cleared on APPLY_* transitions.
- `fdf4974` — Refactored all transitions to structured sideEffects + emit fields format.

**Validator opportunity:** High value.
- For each state machine transition, verify that every mutable field on the entity is either explicitly set in `sideEffects` or explicitly marked as "unchanged".
- Cross-reference the field list from the domain type with the sideEffects keys.
- Flag transitions where a boolean/flag field is set to `true` on one path but never cleared on the reverse path.

#### Category 6: Architectural/Design Change (5 commits, 5.6%)

Actual design changes — these are the only "genuine" issues.

**All commits:**
- `8e225c6` — Replaced Sec-WebSocket-Protocol auth with Socket.IO native auth. **Wrong technology choice.**
- `3a0584e` — Replaced shared processingAttempt with per-stage enrichAttempt/applyAttempt. **Design bug** — shared counter let one stage consume another's retry budget.
- `56dd9ab` — Removed obsolete status and MARK_OBSOLETE feature entirely. **Scope creep cut.**
- `f595c00` — Aligned envelopes.ts with Socket.IO decision, removed serverSeq/sessionToken. **Cleanup from design change.**
- `ca3257d` — Extracted MAX_PROCESSING_ATTEMPTS to rules.ts. **Refactoring.**

**Validator opportunity:** None. These require human/design judgment.

#### Category 7: Missing Error Codes/Handling (5 commits, 5.6%)

Error types or failure paths not defined.

**All commits:**
- `b002e57` — ENRICH_EXHAUSTED error code missing (retry limit reached, no error to throw).
- `3caaca4` — APPLY_EXHAUSTED error code missing.
- `cfab6df` — All error codes existed but had no "Thrown by" comments (14 error codes affected).
- `fdeeef2` — Chat abort replay semantics undefined in ChatDonePayload.
- `fdb8a28` — sourceHashStale self-heal contract not formalized in invariants.ts.

**Validator opportunity:**
- For each terminal/error state in the state machine, verify a corresponding error code exists in errors.ts.
- For each error code, verify it has a "Thrown by" annotation.
- For each exhaustion path (attempt >= MAX), verify an exhaustion error code exists.

#### Category 8: Broken References (4 commits, 4.4%)

Dangling references, missing imports, broken anchors.

**All commits:**
- `d856eda` — sessionToken ghost references after Socket.IO migration (56 lines of stale references).
- `f76c5fb` — MAX_PROCESSING_ATTEMPTS used but not imported in state-machine.ts.
- `1452954` — Dangling master.md reference in Context section.
- `496ec7f` — Broken anchor to Discussion agent tools section.

**Validator opportunity:** Straightforward.
- Verify all `import` statements resolve to existing exports.
- Verify all markdown anchors/links resolve.
- Verify all constant references (like `MAX_PROCESSING_ATTEMPTS`) have matching imports or definitions.
- Search for terms that were renamed (maintain a rename log) and flag stale references.

#### Category 9: Code-Prose Drift (5 commits, 5.6%)

Prose and TypeScript code describing the same thing differently (structural/semantic disagreements).

**All commits:**
- `0fd8b05` — state-machine.ts referenced timeout config fields that didn't exist in config/schema.ts.
- `a687372` — Prose described watchdog with `processExpiredLeases` but no function contract in code.
- `136bc17` — processing/reset endpoint in prose and REST table but not in commands.ts.
- `f6f9756` — Guard string-to-function mapping missing from guards docstring.
- `c91f05a` — RELEASE transition watchdog trigger unclear between prose and code.

**Validator opportunity:**
- Extract all identifiers from state-machine.ts (config refs, function calls, guard names) and verify they exist in the referenced module.
- Extract all endpoint names from prose REST tables and verify they exist in commands.ts or have explicit "REST-only" annotation.

---

## 4. Root Cause Analysis

### 4.1 No "Done" Definition for Specification Completeness

Every review round finds more "add missing X" and "document Y" items because there's no defined criteria for when a spec file is complete. The documentation/clarification category (27%) is theoretically infinite — you can always document more.

### 4.2 Multi-File Approach Created Inter-File Consistency Tax

The old problem: intra-prose inconsistency within an 800+ line monolith. The new problem: **inter-file inconsistency**. When you rename `serverSeq` to `runSeq` in envelopes.ts, no linter tells you the prose examples still use the old name. Cross-reference (11%) and code-prose drift (6%) categories stem from this.

### 4.3 Review Agents Optimize for Finding, Not Severity Assessment

The agents are configured to **find** problems. Each round they search and find — but a missing side effect and a missing docstring are different severity levels. Insufficient severity filtering leads to long tails of minor documentation gaps being treated as equally important.

### 4.4 The PRD Is Too Detailed for a PRD

5,912 lines with regex validation, branded type runtime assertions, 300ms debounce specs — these are **implementation details**, not product requirements. The larger the specification surface, the more places inconsistency can emerge.

### 4.5 Fixes Generate New Fix Requirements (Whack-a-Mole)

Adding a `processingStage` field means updating all 11 transitions. Adopting Socket.IO auth means removing `sessionToken` references everywhere. Each fix creates 2-3 new potential inconsistencies. This is the fundamental reason the issue count doesn't reach zero quickly.

---

## 5. Improvement Proposals

### 5.1 Completeness Checklist per File Type (Immediate)

Define explicit "done" criteria for each file type:

**State machine transitions:**
- [ ] Guard specified (or explicitly "none")
- [ ] sideEffects complete (every mutable field addressed)
- [ ] emit event specified
- [ ] Error code defined if failure path

**Protocol files (commands, events, dto):**
- [ ] Every command has ACK type in AckPayloadMap
- [ ] Every DTO uses branded types where applicable
- [ ] Error response shape defined

**Config schema:**
- [ ] Every field referenced in state-machine.ts or rules.ts exists
- [ ] Every field has a default value or is marked required

This turns an open-ended "find more issues" review into a bounded checklist verification.

### 5.2 Static Validator (Medium Effort, High Impact)

**This is the highest-leverage improvement.** Based on the category analysis, a validator could automatically catch ~55-60% of all historical issues.

#### Validation Rules by Priority

**P0 — Cross-Reference Integrity (would catch ~15 commits)**
1. Every identifier in state-machine.ts `guard`, `sideEffects`, `emit` fields resolves to an existing definition
2. Every config field referenced in any .ts file exists in config/schema.ts
3. Every constant (e.g., `MAX_PROCESSING_ATTEMPTS`) is imported where used
4. Every enum/union value used in code files matches the canonical definition in types.ts
5. Markdown anchors and file references resolve

**P1 — Structural Completeness (would catch ~20 commits)**
1. Every state machine transition has: guard (or explicit "none"), sideEffects, emit
2. Every mutable field on WorkItem type appears in every transition's sideEffects (set or explicitly "unchanged")
3. Every CommandName has a corresponding AckPayloadMap entry
4. Every error state / exhaustion path has a corresponding error code in errors.ts
5. Every error code has a "Thrown by" annotation

**P2 — Type Consistency (would catch ~10 commits)**
1. No inline literal unions (`"raw" | "enriched"`) where a named type exists (ItemStatus)
2. DTOs use branded types (ItemId, not string) for identity fields
3. Schema types (INTEGER, TEXT) match corresponding TypeScript types (number, string, boolean)

**P3 — Prose-Code Alignment (would catch ~5 commits, harder to implement)**
1. Identifiers in prose code blocks match actual code file exports
2. Field names in prose API tables match DTO definitions
3. Prose examples use current terminology (detect renamed terms)

#### Implementation Approach

The validator should work as a **lint-like CLI tool** that:
- Parses .ts files to extract: type names, field names, enum values, imports, exports, state machine structure
- Parses .md files to extract: code blocks, inline code references, anchor links, API tables
- Runs validation rules and outputs findings with file:line references
- Can be integrated into the review workflow (run before CraftPRD Review)

### 5.3 PRD vs Implementation Boundary (Strategic)

Move implementation-level detail out of the PRD:

| Stays in PRD | Moves to implementation backlog |
|--------------|--------------------------------|
| State machine (states, transitions, guards) | Regex validation details (e.g., idempotency key format) |
| Domain model (types, enums) | Branded type runtime checks |
| API contract (endpoints, payload shapes) | Debounce timing (300ms) |
| Invariants (business rules) | Docstrings ("concurrent-safe guarantee") |
| Error taxonomy (what errors exist) | "Thrown by" comments |
| Config schema (what config exists) | Default values for non-critical config |

This would reduce PRD surface area by ~30-40%, proportionally reducing the consistency maintenance burden.

### 5.4 Review Strategy Change

**Current:** "Find everything" → infinite cycle

**Proposed:** "Find what would mislead implementation"

Severity filter for review findings:
- **Critical**: Design flaw, missing state/transition, wrong API contract → **FIX**
- **Important**: Missing guard, missing side effect, cross-ref inconsistency → **FIX**
- **Minor**: Missing docstring, clarification, "document X" → **DO NOT FIX in PRD**; track as implementation notes or let implementation reveal what actually needs clarification

This would eliminate ~30% of fixes (most of the "Missing documentation/clarification" category).

### 5.5 Declare "Good Enough" and Start Implementation

The trend is decreasing (11.2 → 5.3 fixes/round). The remaining fixes are 94% completeness, not design. At this point, the PRD is good enough to **begin implementation**. Implementation will naturally surface which specifications were actually ambiguous vs which "missing documentation" items were clear enough from context.

---

## 6. Static Validator — Detailed Spec Input

This section provides concrete data for building the validator.

### 6.1 File Dependency Graph

```
types.ts ← (imported by all other .ts files)
  ↓
state-machine.ts → references: guards (rules.ts), config (config/schema.ts),
                    errors (errors.ts), types (types.ts)
  ↓
rules.ts → references: types (types.ts), config (config/schema.ts)
  ↓
invariants.ts → references: types (types.ts)
  ↓
commands.ts → references: types (types.ts)
  ↓
events.ts → references: types (types.ts), commands.ts (CommandName)
  ↓
dto.ts → references: types (types.ts), events.ts (EventName)
  ↓
envelopes.ts → references: commands.ts, events.ts, dto.ts
  ↓
errors.ts → references: types (types.ts)
  ↓
config/schema.ts → standalone (Zod schemas)
  ↓
db/schema.ts → references: types (types.ts)
  ↓
agent contracts (runner, producer, enricher, apply, discussion)
    → reference: types (types.ts), dto.ts, commands.ts
```

### 6.2 Cross-Reference Patterns to Validate

| Source | Target | What to check |
|--------|--------|---------------|
| state-machine.ts `emit` fields | events.ts `ServerEventName` | Every emit value is a valid event name |
| state-machine.ts `guard` fields | Actual guard function signatures | Every guard string maps to a defined function |
| state-machine.ts timeout refs | config/schema.ts fields | Every timeout config exists in schema |
| state-machine.ts constant refs | Imports from rules.ts/types.ts | Every constant is imported |
| commands.ts `CommandName` | envelopes.ts `WsCommandEnvelope` | Every command has an envelope type |
| commands.ts `CommandPayloadMap` | dto.ts `AckPayloadMap` | Every command key has an ACK payload |
| events.ts inline unions | types.ts canonical types | No inline literals where named type exists |
| errors.ts error codes | state-machine.ts error transitions | Every error path has a code |
| db/schema.ts columns | types.ts fields | Schema columns match domain type fields |
| prose API tables | commands.ts + dto.ts | Endpoint names and payload fields match |
| prose code examples | Actual code exports | Identifiers in examples are current |

### 6.3 State Machine Transition Template

Every transition should be validated against this structure:

```typescript
// TRANSITION_NAME
// from: StateA → to: StateB
{
  guard: "guardFunctionName",    // REQUIRED (or explicit "none")
  sideEffects: {
    // Every mutable field on WorkItem must appear here
    status: "new_status",
    processingStage: "new_stage",
    // ... every other mutable field: explicit value or "unchanged"
  },
  emit: "event.name",           // REQUIRED: must be valid ServerEventName
  errorCode: "ERROR_CODE",      // REQUIRED if this is a failure transition
  notes: "..."                  // OPTIONAL
}
```

### 6.4 Mutable Fields Checklist (for Side Effect Validation)

These fields should be verified in EVERY transition's sideEffects:

From the WorkItem type:
- `status` (always changes)
- `processingStage`
- `enrichAttempt` / `applyAttempt`
- `sourceHashStale`
- `conflictNote`
- `applyShouldAbort`
- `activeReviewItemId` (run-level)
- `leaseExpiresAt`

The validator should extract this list from the actual WorkItem type definition and check each transition covers all fields.

---

## 7. Appendix: All Categorized Commits

### Missing Types/Interfaces (16)

| Commit | Description |
|--------|-------------|
| `92d9800` | Add ServerMessage discriminated union to envelopes.ts |
| `42a9192` | Add ProcessingResetResponse type for REST-only admin endpoint |
| `a78e33f` | Add SourceResponseDTO for GET /source endpoint |
| `e00ea8b` | Add RestErrorResponse type for consistent REST/WS error format |
| `2ba6911` | Add typed ACK payloads and AckPayloadMap for all commands |
| `a8435d2` | Add RuntimeState interface for runSecret lifecycle |
| `3c5430d` | Add IdempotencyKey branded type with prefix validation |
| `ad30e12` | Use ItemId branded type in command/event/DTO payloads |
| `7000f6b` | Add chat.chunk and chat.done event definitions to events.ts |
| `a68afb5` | Add ApplyAbortError and checkApplyShouldAbort contract |
| `9a1fdfd` | Enforce expectedVersion via discriminated union |
| `740bff3` | Add category field to DecisionLogEntry |
| `60622b0` | Add PRODUCER_STARTUP_GRACE_MS to code source of truth |
| `de6181d` | Add missing category column to prd_review_decisions |
| `9ec0ef4` | Clarify expectedVersion optionality, add db/schema.ts |
| `01f41bd` | Define persona exhaustion behavior with shouldStopProducer |

### Missing Guards/Validations (9)

| Commit | Description |
|--------|-------------|
| `027ac87` | Add processingAttempt guard to ENRICH_START transitions |
| `4574661` | Add expectedVersion guard to DECISION_ACCEPT and DECISION_REJECT |
| `cd98f9f` | Add canAccept guard function and guard contract comment |
| `7dfa888` | Add CLAIM composite guard and INITIAL_STATUS declaration |
| `df167d3` | Add max_enrichers concurrency check to enricher pseudocode |
| `77cada1` | Add full regex + length validation to asIdempotencyKey |
| `5bc2fcf` | Add assertBindAddress runtime assertion for localhost-only binding |
| `050c124` | Correct processing/reset precondition logic |
| `34aec89` | Remove chat.send from AckPayloadMap (incorrectly included) |

### Cross-Reference Inconsistency (10)

| Commit | Description |
|--------|-------------|
| `520bc75` | Align state machine emit fields with ClientEventName |
| `68ee9bc` | Rename ProducerRuntime.currentPersona to persona |
| `6220367` | Replace serverSeq with runSeq in all message examples |
| `9df091d` | Replace inline literal unions with imported domain types |
| `d91113a` | Make transcriptMessages required with [] default |
| `70cfe97` | Correct INTEGER to boolean type cast in schema and invariants |
| `21a53e1` | Clarify AckPayloadMap extends (not mirrors) CommandPayloadMap |
| `25bcbe1` | Strengthen bindAddress prose to match literal type constraint |
| `26ec386` | Fix ChatDonePayload streamInterrupted examples to include finalText |
| `3ec9951` | Add HTTP success status codes and ACK type refs to REST endpoint table |

### Missing Documentation/Clarification (24)

| Commit | Description |
|--------|-------------|
| `aec363d` | Document implicit itemVersion/updatedAt side effects |
| `2afa86e` | Clarify REST workerId enforcement vs trust policy |
| `2e4fd84` | Clarify REST vs WS error response shape in dto.ts |
| `bd8f99e` | Add PROCESSING_RESET comment block to state-machine.ts |
| `36c205a` | Document APPLY_SOURCE_UNAVAILABLE in PRD prose |
| `d4d044e` | Document REST workerId spoofing as explicit V1 limitation |
| `9bc2144` | Document applyShouldAbort mechanism in state-machine.ts |
| `c97d47e` | Clarify producerRuntime null semantics in docstring |
| `f6f2e16` | Document concurrent-safe guarantee for abortAndCleanup |
| `032d1a8` | Specify FIFO resend order for reconnect outbox replay |
| `4d54997` | Document applyShouldAbort early abort in APPLY_SOURCE_CHANGED |
| `b3017bf` | Clarify multi-tab safe scope (browser tabs via WS only) |
| `5e7f716` | Document abortAndCleanup requirement for in_review exits |
| `58df9e7` | Cross-reference sourceHashStale self-heal in APPLY_SUCCESS notes |
| `8f8588b` | Document localhost bind enforcement in bindAddress docstring |
| `82cb7e7` | Clarify RuntimeState vs DB-persisted run state in docstring |
| `9bba742` | Document ENRICH_START concurrency safety in transition notes |
| `97c0378` | Document V1 localhost-only bind constraint in config schema |
| `d5e8908` | Document enricher mid-flight source loss recovery via ENRICH_TIMEOUT |
| `d9ec979` | Clarify processingAttempt reset-then-increment sequence |
| `70d88be` | Document hashAtCreate purpose and lifecycle in PrdReviewContext |
| `5129df1` | Clarify runner.ts is spec-level contract file, not TODO |
| `7eb5073` | Clarify ensureSourceHashFresh is server-layer delegate, not TODO |
| `d19a34c` | Clarify V1 scope: single-person, not single-actor |

### Missing Side Effects (10)

| Commit | Description |
|--------|-------------|
| `33eee93` | Clear conflictNote on all needs_reenrich paths |
| `3caa91b` | Add sourceHashStale clear to APPLY_SUCCESS sideEffects |
| `5f6d065` | Add processingStage to sideEffects in all 11 transitions |
| `0973262` | Add applyShouldAbort clearing to all APPLY_* transitions |
| `40545ab` | Add emit event names to timeout transition notes |
| `fdf4974` | Refactor transitions to structured sideEffects + emit fields |
| `0ef0fe6` | Specify watcher debounce as 300ms trailing-edge with hash dedup |
| `48e6979` | Document idem_ prefix requirement for IdempotencyKey |
| `b3e3227` | Clarify REST workerId spoofing as convention, not enforcement |
| `9ec00da` | Specify rename failure, orphan temp file cleanup in atomic write |

### Architectural/Design Change (5)

| Commit | Description |
|--------|-------------|
| `8e225c6` | Replace Sec-WebSocket-Protocol auth with Socket.IO auth |
| `3a0584e` | Replace shared processingAttempt with per-stage enrichAttempt/applyAttempt |
| `56dd9ab` | Remove obsolete status and MARK_OBSOLETE feature entirely |
| `f595c00` | Align envelopes.ts with Socket.IO decision, remove serverSeq/sessionToken |
| `ca3257d` | Extract MAX_PROCESSING_ATTEMPTS to rules.ts |

### Missing Error Codes/Handling (5)

| Commit | Description |
|--------|-------------|
| `b002e57` | Add ENRICH_EXHAUSTED error code to errors.ts |
| `3caaca4` | Add APPLY_EXHAUSTED error code to errors.ts |
| `cfab6df` | Add "Thrown by" comments to all error codes in errors.ts |
| `fdeeef2` | Clarify chat abort replay semantics in ChatDonePayload |
| `fdb8a28` | Formalize sourceHashStale self-heal contract in invariants.ts |

### Broken References (4)

| Commit | Description |
|--------|-------------|
| `1452954` | Remove dangling master.md reference from Context section |
| `f76c5fb` | Add missing MAX_PROCESSING_ATTEMPTS import to state-machine |
| `496ec7f` | Fix broken anchor to Discussion agent tools section |
| `d856eda` | Remove sessionToken ghost references, align with Socket.IO |

### Code-Prose Drift (5)

| Commit | Description |
|--------|-------------|
| `f6f9756` | Add explicit guard string to function mapping in guards docstring |
| `c91f05a` | Clarify RELEASE transition watchdog trigger in notes |
| `a687372` | Add watchdog processExpiredLeases contract to state-machine |
| `0fd8b05` | Add missing timeout config fields referenced by state-machine |
| `136bc17` | Document REST-only endpoint processing/reset in commands.ts |
