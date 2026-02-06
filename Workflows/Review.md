---
description: Review a PRD or structured spec for inconsistencies using parallel expert agents
argument-hint: <prd-path> [<scope>]
allowed-tools: Read, Write, Glob, Grep, AskUserQuestion, Task
---

# Review — Parallel Consistency Check for PRD or Structured Spec

Launch 6 parallel expert agents to review a PRD (monolithic or structured) for inconsistencies, gaps, and technical issues. Each agent focuses on a specific consistency dimension with fresh context. Findings are merged, deduplicated, and reported by severity.

## Variables

PRD_PATH: $1 (path to the main PRD or design doc)
SCOPE: $2 (optional: "full" | "quick" — default: "full")
AGENT_MODEL: sonnet (for review agents — fast, thorough, cost-effective)

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide a path to the PRD or design doc."
- This workflow is **read-only** — it does NOT modify any files. It produces a findings report.
- Launch **all 6 agents in parallel** using the Task tool with `subagent_type: "general-purpose"`.
- If `SCOPE` is "quick", launch only agents 1-3 (the most impactful checks).
- Each agent gets the PRD path and any referenced code file paths. The agent reads the files itself.
- Focus on **actionable findings** — things that would cause bugs or confusion during implementation.

## Workflow

### 1. Read and Detect Structure

- Read the PRD at `PRD_PATH`.
  - If not found → STOP with "PRD not found: {PRD_PATH}"
- Detect the PRD type:
  - **Monolithic:** Single large markdown file (>500 lines, contains code blocks, schemas, etc.)
  - **Structured:** Design doc that references external code files (contains "See `packages/...`" or similar references)
- If structured, find all referenced code file paths using Glob/Grep.
- Build a file list: `[PRD_PATH, ...referenced_code_files]`

Report to user:
```
Starting review of: <PRD_PATH>
Type: <Monolithic | Structured>
<If structured:> Referenced files: <count> files found
Scope: <full | quick>
Launching <6 | 3> parallel review agents...
```

### 2. Launch Parallel Review Agents

Launch all agents in a **single message** (parallel Task tool calls). Each agent receives the same file list but focuses on its specific dimension.

#### Agent 1: Type Consistency Agent
```
You are a Type Consistency expert reviewing a PRD for type-level inconsistencies.

Read ALL of these files thoroughly:
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

Read ALL of these files thoroughly:
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

Read ALL of these files thoroughly:
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

Read ALL of these files thoroughly:
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

Read ALL of these files thoroughly:
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

Read ALL of these files thoroughly:
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

### 3. Collect and Merge Results

After all agents complete:

1. **Collect** all findings from all 6 agents.
2. **Deduplicate** — if two agents found the same issue (same location, same problem), merge into one finding, noting which agents found it.
3. **Number** findings sequentially (#1, #2, #3...).
4. **Group** by severity: Critical → Important → Minor.

### 4. Generate Report

Write the merged report to the user (and optionally to a file if the PRD is large).

## Report

```
PRD Review: <PRD name>
Type: <Monolithic | Structured>
Scope: <full | quick>
Agents: <6 | 3> launched, <N> returned findings

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
```

After the report, suggest: "To fix these findings, edit the PRD directly. After fixing, run the Review workflow again to verify."
