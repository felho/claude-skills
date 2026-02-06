---
description: Review a PRD or structured spec for inconsistencies using parallel expert agents. Adapts to document size.
argument-hint: <prd-path> [<scope>]
allowed-tools: Read, Write, Glob, Grep, AskUserQuestion, Task
---

# Review — Parallel Consistency Check for PRD or Structured Spec

Launch parallel expert agents to review a PRD (monolithic or structured) for inconsistencies, gaps, and technical issues. Each agent focuses on a specific consistency dimension with fresh context. Adapts reading strategy based on document size to stay within context limits. Findings are merged, deduplicated, and reported by severity.

## Variables

PRD_PATH: $1 (path to the main PRD or design doc)
SCOPE: $2 (optional: "full" | "quick" — default: "full")
AGENT_MODEL: sonnet (for review agents — fast, thorough, cost-effective)

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide a path to the PRD or design doc."
- This workflow is **read-only** — it does NOT modify any files. It produces a findings report.
- Launch **all 7 agents in parallel** using the Task tool with `subagent_type: "general-purpose"`.
- If `SCOPE` is "quick", launch only agents 1-3 (the most impactful checks).
- Each agent gets the PRD path and any referenced code file paths. The agent reads the files itself.
- Focus on **actionable findings** — things that would cause bugs or confusion during implementation.
- **Context Budget Rule:** The orchestrating agent (you) must **NEVER** read the full PRD with the Read tool. Only scan structure using Grep. Each review agent reads file content in its own independent context window. Reading a large PRD here leaves no room for collecting 7 agents' results.

## Workflow

### 1. Scan Structure (Do NOT Read Full File)

> **Do NOT use Read on the full PRD.** Reading a large file here consumes the context needed for collecting agent results. Agents read files in their own context windows.

1. **Verify file exists**: `Read` with `limit: 30` — read ONLY the first 30 lines for title and overview context.
   - If not found → STOP with "PRD not found: {PRD_PATH}"
2. **Count total lines**: `Grep` with pattern `"."` and `output_mode: "count"` on the PRD file.
3. **Build section map**: `Grep` with pattern `^#{1,3} ` and `output_mode: "content"` with `-n: true` to get all headings with line numbers.
4. **Detect structure type**:
   - **Monolithic:** Single markdown file, no external code references
   - **Structured:** References external files (look for paths like `packages/`, `src/`, backtick-quoted `.ts`/`.md` file paths)
5. **If structured**: Find referenced file paths using Grep/Glob. Count lines in each.
6. **Build FILE_LIST**: `[PRD_PATH, ...referenced_code_files]`
7. **Build CODE_FILE_MAP** (structured PRDs only): For each referenced code file, record:
   - File path
   - Line count
   - Brief content hint (infer from filename: `types.ts` → "type/enum definitions", `state-machine.ts` → "state transitions", `errors.ts` → "error codes", etc.)
8. **Calculate TOTAL_LINES** across all files. Also calculate **CODE_LINES** (sum of code file lines only) and **PRD_LINES** (PRD file lines only).
9. **Select review mode**:
   - **Standard** (TOTAL_LINES < 1000): Agents read full files directly
   - **Targeted** (TOTAL_LINES ≥ 1000): Agents receive section map + code file map and read selectively

Report to user:
```
Starting review of: <PRD_PATH>
Type: <Monolithic | Structured>
Lines: <TOTAL_LINES> → <standard | targeted> mode
<If structured:> Referenced files: <count> files found
Scope: <full | quick>
Launching <7 | 3> parallel review agents...
```

### 2. Launch Parallel Review Agents

Launch all agents in a **single message** (parallel Task tool calls). Each agent focuses on its specific dimension.

**Mode-specific prompt construction:**

- **Standard mode** (< 1000 lines): Use agent prompts as written below — agents read full files.
- **Targeted mode** (≥ 1000 lines): **Prepend** the Targeted Reading Prefix to each agent prompt before the agent's own instructions.

#### Targeted Reading Prefix

When TOTAL_LINES ≥ 1000, prepend this block to every agent prompt (fill in variables from Step 1).

**For monolithic PRDs** (no companion code files):

```
CONTEXT: This is a large document ({TOTAL_LINES} lines).
Use targeted reading to stay within context limits.

Section map of {PRD_PATH}:
{SECTION_MAP_FROM_STEP_1}

YOUR PRIORITY KEYWORDS: {AGENT_KEYWORDS}

READING STRATEGY:
1. Identify sections from the map above that match your priority keywords.
2. Read those sections using the Read tool with offset and limit parameters.
   Example: Read(file_path: "...", offset: 89, limit: 90) for lines 89-178.
3. If a finding references another section, read that section too.
4. Stay under {READING_BUDGET} lines total read.
5. Return at most 10 findings, highest severity first.
```

**For structured PRDs** (with companion code files):

```
CONTEXT: This is a large structured document ({TOTAL_LINES} lines across {FILE_COUNT} file(s)).
The prose design doc references companion code files that contain the AUTHORITATIVE formal definitions (types, state machine, error codes, etc.).
Use targeted reading to stay within context limits.

PRD section map of {PRD_PATH}:
{SECTION_MAP_FROM_STEP_1}

Companion code files (SOURCE OF TRUTH for formal definitions):
{CODE_FILE_MAP}

YOUR PRIORITY KEYWORDS: {AGENT_KEYWORDS}

READING STRATEGY — CODE FILES FIRST:
1. Read ALL companion code files listed above IN FULL. They are small (typically 100-400 lines each) and contain the authoritative definitions. This is your FIRST and MOST IMPORTANT step.
2. Then identify PRD sections from the map above that match your priority keywords.
3. Read those PRD sections using the Read tool with offset and limit parameters.
   Example: Read(file_path: "...", offset: 89, limit: 90) for lines 89-178.
4. If a finding references another section, read that section too.
5. Stay under {READING_BUDGET} lines total read from the PRD file. Code files do NOT count toward this budget.
6. Return at most 10 findings, highest severity first.

CRITICAL — VERIFY BEFORE REPORTING:
Before reporting ANY finding about missing, incomplete, or inconsistent types/transitions/errors/definitions:
- CHECK the companion code files you read in step 1. The code files are the SOURCE OF TRUTH.
- If a code file contains the correct, complete definition and the PRD prose references it, this is NOT a finding.
- Only report an issue if: (a) it is genuinely missing from BOTH prose AND code files, or (b) the prose CONTRADICTS the code file, or (c) the issue is about the prose-code relationship itself (e.g., prose references a type that doesn't exist in the code).
```

**`{CODE_FILE_MAP}` format** (built from Step 1):
```
- {file_path} ({line_count} lines) — {content_hint}
```
Example:
```
- packages/domain/types.ts (269 lines) — type/enum definitions
- packages/domain/state-machine.ts (327 lines) — state transitions, guards, side effects
- packages/protocol/errors.ts (102 lines) — error codes, HTTP status mapping
```

**`{READING_BUDGET}` calculation**:
- **Monolithic**: `min(2000, TOTAL_LINES * 0.5)`
- **Structured**: `min(2000, PRD_LINES * 0.5)` — budget applies ONLY to PRD lines, code files are read in full and don't count toward budget.

**Agent priority keywords** (for `{AGENT_KEYWORDS}` in targeted prefix):

| Agent | Keywords |
|-------|----------|
| Type Consistency | type, interface, schema, model, data, enum, definition, payload, field |
| State Machine | state, machine, lifecycle, transition, status, flow, recovery, watchdog, guard |
| API Contract | api, endpoint, route, rest, websocket, event, command, payload, error, response |
| Cross-Reference | *(all sections — check "see section" refs, dangling links, field name consistency)* |
| Completeness | *(all sections — scan for TODO, TBD, undefined, unspecified, gap, placeholder)* |
| Architectural | architect, design, decision, boundary, security, scope, v1, v2, trade-off, principle |
| Build vs Reuse | technolog, framework, library, tool, stack, custom, protocol, implement |

---

#### Agent 1: Type Consistency Agent
```
You are a Type Consistency expert reviewing a PRD for type-level inconsistencies.

Files to analyze:
<FILE_LIST>

Then analyze:
- Are the same types/interfaces defined identically everywhere they appear?
- Same field with different types in different sections or files?
- Missing fields in one definition that exist in another?
- Enum values that differ between sections (e.g., status values in data model vs API vs events)?
- TypeScript interfaces in markdown that contradict each other?
- Payload schemas that don't match their TypeScript interface definitions?
- Union types that include or exclude different members in different places?

For each finding, provide:
1. **Title** — short description
2. **Severity** — Critical (would cause bug/data loss), Important (would cause confusion), Minor (cosmetic)
3. **Where** — exact section names, line numbers, or file paths where the inconsistency appears
4. **Description** — what's inconsistent and why it matters
5. **Recommendation** — specific fix

Return findings as a numbered list. If no findings, say "No type consistency issues found."
```

#### Agent 2: State Machine Completeness Agent
```
You are a State Machine expert reviewing a PRD for state machine completeness.

Files to analyze:
<FILE_LIST>

Then analyze:
- Is every state reachable from the initial state?
- Does every non-final state have at least one outgoing transition?
- Are ALL transitions documented with guards (preconditions) and side effects?
- Do recovery/timeout/watchdog transitions match the main state diagram?
- Are there transitions described in prose (e.g., in recovery sections, error handling, API preconditions) that are NOT in the state machine diagram or definition?
- Are status enum values consistent between the state machine definition, API sections, and event definitions?
- If an item can reach a state through multiple paths, are the side effects consistent?

For each finding, provide:
1. **Title** — short description
2. **Severity** — Critical / Important / Minor
3. **Where** — exact section names, line numbers, or file paths
4. **Description** — what's missing or inconsistent
5. **Recommendation** — specific fix

Return findings as a numbered list. If no state machine is defined, say "No state machine found in this PRD." If found but complete, say "State machine is complete — no issues found."
```

#### Agent 3: API Contract Alignment Agent
```
You are an API Contract expert reviewing a PRD for API-level inconsistencies.

Files to analyze:
<FILE_LIST>

Then analyze:
- For each REST endpoint or WebSocket command:
  - Does the payload schema match the TypeScript interface?
  - Are preconditions/guards consistent with the state machine?
  - Are error responses documented for all failure cases?
  - Are HTTP status codes or error codes specified?
- Do event names in any event catalog match the type definitions?
- Are command acknowledgment and error response formats consistent?
- Do internal vs client-facing event classifications match throughout?
- Are idempotency requirements specified consistently?
- Do endpoint descriptions match the actual payload definitions?

For each finding, provide:
1. **Title** — short description
2. **Severity** — Critical / Important / Minor
3. **Where** — exact section names, line numbers, or file paths
4. **Description** — what's inconsistent
5. **Recommendation** — specific fix

Return findings as a numbered list. If no API contract is defined, say "No API contract found." If found but consistent, say "API contract is consistent — no issues found."
```

#### Agent 4: Cross-Reference Validity Agent
```
You are a Cross-Reference expert reviewing a PRD for broken or inconsistent references.

Files to analyze:
<FILE_LIST>

Then analyze:
- Do all "see section X" or "as described in Y" references point to existing sections?
- Are referenced code files present and do they match the prose description?
- Are field names consistent across data model, API, event, and state machine sections?
- Are concepts introduced in one section and used in another with the same meaning?
- Are there sections that reference decisions from a "Design Decisions" log — do those decisions match the actual implementation described elsewhere?
- Are there dangling references (pointing to content that was moved or deleted)?

For each finding, provide:
1. **Title** — short description
2. **Severity** — Critical / Important / Minor
3. **Where** — the reference location AND the target (or missing target)
4. **Description** — what's broken or inconsistent
5. **Recommendation** — specific fix

Return findings as a numbered list. If all references are valid, say "All cross-references are valid — no issues found."
```

#### Agent 5: Completeness Gaps Agent
```
You are a Completeness expert reviewing a PRD for missing definitions, unspecified behavior, and gaps.

Files to analyze:
<FILE_LIST>

Then analyze:
- Are there entities, types, or concepts mentioned but never formally defined?
- Are there state transitions without documented side effects?
- Are there error scenarios without specified handling behavior?
- Are default values specified for all fields that need them?
- Are there missing TypeScript interfaces that other sections reference?
- Are there "TODO", "TBD", "FIXME", or placeholder markers that need resolution?
- Are there scenarios where behavior is ambiguous (two valid interpretations)?
- Are there edge cases that are mentioned but not fully specified?

For each finding, provide:
1. **Title** — short description
2. **Severity** — Critical / Important / Minor
3. **Where** — exact section names, line numbers, or file paths where the gap exists
4. **Description** — what's missing and why it matters for implementation
5. **Recommendation** — what needs to be specified

Return findings as a numbered list. If the PRD is complete, say "No completeness gaps found."
```

#### Agent 6: Architectural Consistency Agent
```
You are an Architectural Consistency expert reviewing a PRD for high-level design issues.

Files to analyze:
<FILE_LIST>

Then analyze:
- Are V1 vs future/V2 boundaries explicit and consistent throughout?
- Are there assumptions that should be documented as explicit decisions?
- Are there design decisions that contradict each other?
- Is the security model consistent (auth tokens, session management, permission checks)?
- Are concurrency/locking concerns addressed consistently (e.g., mutex usage, CAS checks)?
- Are there sections that describe the same concept with different semantics?
- Is the data flow consistent end-to-end (producer → processing → output)?
- Are there "accepted risks" or "V1 trade-offs" that contradict other sections?

For each finding, provide:
1. **Title** — short description
2. **Severity** — Critical / Important / Minor
3. **Where** — exact section names, line numbers, or file paths
4. **Description** — what's inconsistent and the architectural impact
5. **Recommendation** — specific fix or decision needed

Return findings as a numbered list. If architecturally consistent, say "No architectural consistency issues found."
```

#### Agent 7: Build vs Reuse Agent
```
You are a Technology Strategy expert reviewing a PRD for "reinventing the wheel" — areas where the PRD describes custom implementations for problems that existing, mature frameworks or libraries already solve.

Files to analyze:
<FILE_LIST>

Then analyze:
- Are there custom protocol designs (message ordering, reconnection, gap recovery, correlation IDs) where existing tools like Socket.IO, tRPC, or GraphQL subscriptions would suffice?
- Are there custom retry/timeout/abort patterns repeated across multiple agents or components where a shared abstraction or library (e.g., p-retry, AbortController patterns) would eliminate boilerplate?
- Are there hand-rolled state machines in prose where XState, Robot, or a formal transition table would provide type safety and visualization?
- Are there custom validation schemas described in markdown where zod, yup, or similar runtime validation libraries would enforce them?
- Are there custom auth/session mechanisms where established patterns (JWT, session middleware, Passport) would be more robust?
- Are there custom caching, queuing, or scheduling implementations where existing solutions (BullMQ, node-cron, lru-cache) exist?
- Are there custom realtime/streaming implementations where SSE, WebSocket libraries, or framework-level solutions handle the transport?
- For each finding: is this area actually CORE to the product (unique differentiator), or is it COMMODITY infrastructure that shouldn't consume spec/implementation effort?

For each finding, provide:
1. **Title** — what's being reinvented
2. **Severity** — Important (significant spec/implementation savings) or Minor (small savings)
3. **Where** — exact sections in the PRD that describe the custom implementation
4. **Lines of spec affected** — approximate line count that would shrink or disappear
5. **Existing alternatives** — 2-3 specific frameworks/libraries that solve this, with brief trade-offs
6. **Recommendation** — which alternative to consider and why

Return findings as a numbered list. If the PRD appropriately uses existing tools for non-core concerns, say "No reinventing-the-wheel issues found — technology choices are appropriate."
```

### 3. Collect and Merge Results

After all agents complete:

1. **Collect** all findings from all 7 agents.
2. **Deduplicate** — if two agents found the same issue (same location, same problem), merge into one finding, noting which agents found it.
3. **Number** findings sequentially (#1, #2, #3...).
4. **Group** by severity: Critical → Important → Minor.

### 4. Generate Report

Write the merged report to the user (and optionally to a file if the PRD is large).

## Report

```
PRD Review: <PRD name>
Type: <Monolithic | Structured>
Lines: <TOTAL_LINES> (<standard | targeted> mode)
Scope: <full | quick>
Agents: <7 | 3> launched, <N> returned findings

---

## Summary

<1-3 sentence high-level assessment>

The most critical issues are: <1-2 sentence summary of critical findings, if any>

| Severity | Count |
|----------|-------|
| Critical | <N> |
| Important | <N> |
| Minor | <N> |
| **Total** | **<N>** |

---

## Findings

### #1 — <Finding title> [Critical]

<Description of the inconsistency or issue>

**Where:** <section/file and line reference>
**Found by:** <agent name(s)>
**Impact:** <what goes wrong if not fixed>
**Recommendation:** <specific fix>

---

### #2 — <Finding title> [Important]
...

---

## Recommended Fix Order

1. <Most impactful fix first — finding #N>
2. <Second — finding #N>
...

## Agent Summary

| Agent | Findings | Status |
|-------|----------|--------|
| Type Consistency | <N> | Completed |
| State Machine | <N> | Completed |
| API Contract | <N> | Completed |
| Cross-Reference | <N> | Completed |
| Completeness | <N> | Completed |
| Architectural | <N> | Completed |
| Build vs Reuse | <N> | Completed |
```

After the report, suggest: "To fix these findings, edit the PRD directly. After fixing, run the Review workflow again to verify."
