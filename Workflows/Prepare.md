---
description: Create step implementation packet from plan + design doc
argument-hint: <plan> <design-doc> [phase-id/step-id]
allowed-tools: Read, Write, Edit, Glob, AskUserQuestion
# Note: Write/Edit are ONLY for the packet .md file, never for implementation files
---

# Prepare Step Packet

Create a detailed step implementation packet containing all context needed for implementation.

> **⚠️ CRITICAL: This workflow ONLY creates a packet markdown file.**
>
> **Do NOT:**
> - Write code or config files
> - Modify project files
> - Execute any implementation steps
> - Create directories outside the packet location
>
> **Your ONLY output is the packet `.md` file** at the computed path (plus updating plan status).
> Implementation happens in the **Execute** workflow, not here.

## Variables

PLAN_PATH: \$1
DESIGN_DOC_PATH: \$2
STEP_ID: \$3

## Instructions

### Plan Structure Requirements

- **Phase:** H2 heading followed by `<!-- id: phase-id -->`
- **Step:** H3 heading followed by `<!-- id: step-id -->` or `<!-- id: step-id status: in-progress|done -->`
- **Full step ID format:** `{phase-id}/{step-id}`

### Status Logic

- No `status` attribute = todo
- `status: in-progress` = currently being worked on
- `status: done` = completed

### Packet Location Convention

Given plan `plans/foo/bar.md` and step `phase-id/step-id`:
→ Packet path: `plans/foo/bar-steps/phase-id/step-id.md`

### Error Messages (Use Exactly)

- Plan not found: `"Plan file not found: {path}"`
- Design doc not found: `"Design document not found: {path}"`
- No steps in plan: `"No steps found in plan (steps require H3 heading + <!-- id: ... --> comment)"`
- Another step in progress: `"Step {other-step-id} is already in-progress. Finish or abandon it first (remove status: in-progress from the plan)."`
- Multiple in progress: `"Multiple steps in-progress. Fix the plan manually."`
- Step done: `"Step {step-id} is already done. To re-implement, remove status: done from the HTML comment, then run /ManageImpStep prepare."`
- Step not found: `"Step not found: {step-id}"`

## Workflow

### 1. Validate Input Files

- If `PLAN_PATH` is empty → STOP and report: `"Usage: /ManageImpStep prepare <plan> <design-doc> [step-id]"`
- If `DESIGN_DOC_PATH` is empty → STOP and report: `"Usage: /ManageImpStep prepare <plan> <design-doc> [step-id]"`
- Read plan file at `PLAN_PATH`
  - If file not found → STOP with error message
- Read design doc at `DESIGN_DOC_PATH`
  - If file not found → STOP with error message

### 2. Parse Plan Structure

Scan the plan for phases and steps:

<parse-plan>
- Find all H2 headings with `<!-- id: {phase-id} -->` comments → these are phases
- Find all H3 headings with `<!-- id: {step-id} -->` or `<!-- id: {step-id} status: {status} -->` → these are steps
- For each step, determine its parent phase (the most recent H2 before it)
- Build a list of all steps with: full-id, title, status, phase-id
</parse-plan>

If no valid steps found → STOP with "No steps found" error

### 3. Find Step to Prepare

<step-selection>
**If STEP_ID is provided:**
- Find the step matching `STEP_ID` (format: `phase-id/step-id`)
- If not found → STOP with "Step not found" error
- If step has `status: done` → STOP with "Step already done" error
- If step has `status: in-progress`:
  - Check if packet exists at expected location
  - If packet exists → Ask user: "Packet already exists. Overwrite?"
    - If user declines → STOP (no changes)
    - If user accepts → continue to create packet (status remains in-progress)
  - If packet doesn't exist → continue to create packet (status remains in-progress)
- If ANOTHER step has `status: in-progress` → STOP with "Another step in progress" error
- Otherwise → this is the step to prepare

**If STEP_ID is NOT provided:**
- Count steps with `status: in-progress`
- If multiple in-progress → STOP with "Multiple steps in-progress" error
- If exactly one in-progress:
  - Check if packet exists
  - If packet exists → Report step info and packet path, then STOP with message: "Step {step-id} is in progress. Run `/ManageImpStep execute {packet-path}` to continue implementation."
  - If packet missing → this is the step to prepare (will create packet, status remains in-progress)
- If none in-progress → find first step without status attribute (todo) → this is the step to prepare
- If all steps are done → STOP with message: "All steps are complete."
</step-selection>

### 4. Extract Step Content

From the plan, extract for the selected step:
- Full step heading and text until next H2 or H3
- Any checkboxes or task items
- Test/Value annotations
- References to other sections or specs

### 5. Gather Context from Design Document

Read the design document thoroughly. Include ALL sections relevant to this step:

<gather-context>
- Sections directly referenced in the step definition
- Technical specifications for the step's domain
- Error handling requirements
- Validation rules
- Edge cases
- Related configuration or types
- Dependencies and prerequisites
</gather-context>

**Think hard:** What information would an implementer need? Include it even if not explicitly referenced.

### 6. Gather Context from Plan

Look for supporting sections in the plan that apply to this step:

<gather-plan-context>
- Error message guidelines
- Type definitions
- Coding standards
- Related steps (dependencies)
- Cross-references (e.g., "see section X", "as defined in Step Y")
</gather-plan-context>

Include the ACTUAL CONTENT of referenced sections, not just links.

### 7. Create Packet Directory (for the markdown file only)

Compute packet path: `{plan-dir}/{plan-name}-steps/{phase-id}/{step-id}.md`

Create parent directories if needed. This is the ONLY directory you should create.

### 8. Write Packet Markdown File (NOT implementation!)

Create the packet `.md` file with this structure (this is the ONLY file you write):

```markdown
---
step: {phase-id}/{step-id}
plan: {PLAN_PATH}
design-doc: {DESIGN_DOC_PATH}
created: {ISO-8601 timestamp}
---

# Step: {phase-id}/{step-id}

## Overview

{Goal statement from step - what this step accomplishes and why}

## Step Definition

{Full step text from implementation plan, including checkboxes}

## From Design Document

{All relevant sections from design doc - include actual content}

## From Plan (Supporting Sections)

{Content of any supporting sections from the plan - actual content, not references}

## Dependencies

{List of prerequisite steps that must be complete}

## Test Cases

{Specific test scenarios from the step or design doc}

## Acceptance Criteria

{What must be true for the step to be complete - derive from step definition}

## Implementation Notes

{Gotchas, critical notes, exact error messages, edge cases}
```

### 9. Update Plan Status

If step was not already `in-progress`:
- Find the step's `<!-- id: {step-id} -->` comment
- Change to `<!-- id: {step-id} status: in-progress -->`

### 10. STOP Check (Self-Verification)

Before reporting, verify you followed the rules:

- [ ] You created ONLY one new file: the packet `.md` at the computed path
- [ ] You did NOT create/modify any code, config, or project files
- [ ] You did NOT implement the step (no `special-rules.json`, no source files, etc.)
- [ ] The packet contains context TO BE IMPLEMENTED LATER in Execute workflow

If any check fails → you made a mistake. Undo the implementation work and focus only on the packet.

### 11. Report Result

Output:
```
✅ Step prepared: {phase-id}/{step-id}

Title: {step heading}
Packet: {packet-path}

Next: Run `/ManageImpStep execute {packet-path}` to implement.
```

## Report

After successful preparation, always include:
1. Confirmation message with step ID
2. Full path to the created packet
3. Suggested next command

**Remember:** You prepared a PACKET (documentation). You did NOT implement anything. Implementation is the user's next step via `/ManageImpStep execute`.
