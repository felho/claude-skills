---
description: Review a PRD or structured spec for inconsistencies and issues
argument-hint: <prd-path> [<scope>]
allowed-tools: Read, Glob, Grep, AskUserQuestion, Task
---

# Review — Consistency Check for PRD or Structured Spec

Systematically review a PRD (monolithic or structured) for inconsistencies, gaps, and technical issues. Reports findings by severity.

## Variables

PRD_PATH: $1 (path to the main PRD or design doc)
SCOPE: $2 (optional: "full" | "quick" | specific section name — default: "full")

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide a path to the PRD or design doc."
- This workflow is **read-only** — it does NOT modify any files. It produces a findings report.
- For a structured spec (PRD + code files), also scan referenced code files for cross-consistency.
- Focus on **actionable findings** — things that would cause bugs or confusion during implementation.
- Do NOT flag stylistic issues or formatting preferences.
- If `SCOPE` is "quick", limit to the top 3 most impactful checks. If a specific section name, focus only on that section.

## Workflow

### 1. Read and Understand Structure

- Read the PRD at `PRD_PATH`.
  - If not found → STOP with "PRD not found: {PRD_PATH}"
- Detect the PRD type:
  - **Monolithic:** Single large markdown file (>500 lines, contains code blocks, schemas, etc.)
  - **Structured:** Design doc that references external code files (contains "See `packages/...`" references)
- If structured, find and read all referenced code files.

### 2. Build Concept Map

Before checking for issues, build a mental model of the key concepts:
- Entity types and their fields
- Status values and transitions
- Event/command names and payloads
- Error codes and their meanings
- API endpoints and their contracts
- Invariants and assertions

This step ensures the review is thorough — you can't find inconsistencies without understanding the full picture.

### 3. Systematic Review

Run checks in this order. For each check, scan the entire document (and referenced files for structured specs).

<review-checks-loop>

**Check 1: Type consistency**
Are the same types/interfaces defined identically everywhere they appear? Look for:
- Same field with different types in different places
- Missing fields in one definition that exist in another
- Enum values that differ between sections

**Check 2: State machine completeness**
If a state machine is defined (as diagram, prose, or code):
- Is every state reachable?
- Does every state have at least one outgoing transition (except final states)?
- Are all transitions documented with guards and side effects?
- Do recovery/timeout transitions match the main diagram?

**Check 3: API contract alignment**
For each API endpoint or command:
- Does the payload schema match the TypeScript interface?
- Are preconditions/guards consistent with the state machine?
- Are error responses documented for all failure cases?
- Do event names in the catalog match the type definitions?

**Check 4: Cross-reference validity**
- Do all "see section X" references point to existing sections?
- Are referenced code files present and consistent with the prose description?
- Are field names consistent across data model, API, and event sections?

**Check 5: Completeness gaps**
- Are there entities mentioned but never defined?
- Are there transitions without documented side effects?
- Are there error scenarios without handling?
- Are default values specified for all fields that need them?

**Check 6: Scope and boundary clarity**
- Are V1 vs future boundaries explicit?
- Are there "TODO" or "TBD" markers that need resolution?
- Are there assumptions that should be documented as decisions?

</review-checks-loop>

### 4. Classify Findings

For each finding, assign a severity:

| Severity | Criteria |
|----------|----------|
| **Critical** | Would cause a bug or data loss. Must fix before implementation. |
| **Important** | Would cause confusion or wrong implementation. Should fix. |
| **Minor** | Cosmetic or clarification. Nice to fix but not blocking. |

### 5. Generate Report

Present findings grouped by the summary categories, with finding numbers for reference.

## Report

```
PRD Review: <PRD name>
Type: <Monolithic | Structured>
Scope: <full | quick | section-name>

---

## Summary

<1-3 sentence high-level assessment>

### Critical (<count>)
<grouped description of critical findings — what's the common theme?>

### Important (<count>)
<grouped description>

### Minor (<count>)
<grouped description>

---

## Findings

### #1 — <Finding title> [<severity>]

<Description of the inconsistency or issue>

**Where:** <section/file and line reference>
**Impact:** <what goes wrong if not fixed>
**Recommendation:** <specific fix>

---

### #2 — <Finding title> [<severity>]
...

---

## Recommended Fix Order

1. <Most impactful fix first — finding #N>
2. <Second — finding #N>
...
```

After the report, suggest: "To fix these findings, edit the PRD directly. After fixing, run the Review workflow again to verify."
