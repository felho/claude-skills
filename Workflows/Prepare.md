---
description: Create step implementation packet from plan + design doc
argument-hint: <plan> <design-doc> [phase-id/step-id] [--ahead N] [--auto-check]
allowed-tools: Read, Write, Edit, Glob, AskUserQuestion, Task
# Note: Write/Edit are ONLY for the packet .md file, never for implementation files
# Note: Task is for parallel prepare-ahead agents and auto-check loop agents
hooks:
  PreToolUse:
    - matcher: "Read"
      hooks:
        - type: command
          command: "uv run $HOME/.claude/hooks/validators/ManageImpStep/design-doc-depth-guard.py"
  PostToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: "uv run $HOME/.claude/hooks/validators/ManageImpStep/packet-structure-validator.py"
  Stop:
    - hooks:
        - type: command
          command: "uv run $HOME/.claude/hooks/validators/ManageImpStep/packet-coverage-validator.py"
        - type: command
          command: "uv run $HOME/.claude/hooks/validators/ManageImpStep/cross-step-consistency-validator.py"
---

# Prepare Step Packet

Create a detailed step implementation packet containing all context needed for implementation.

> **⚠️ CRITICAL: This workflow ONLY creates a packet markdown file.**
>
> **Do NOT:**
>
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
STEP_ID: \$3 (may be absent if flags are used without a step ID)
AHEAD_COUNT: from `--ahead N` flag (default: not set — normal mode)
AUTO_CHECK: from `--auto-check` flag (default: false)

**Flag parsing:** Scan all arguments for `--ahead N` (integer N) and `--auto-check` (boolean). Flags are combinable and order-independent. Remove flags from positional arguments before processing.

## Instructions

### Plan Structure Requirements

- **Phase:** H2 heading followed by `<!-- id: phase-id -->`
- **Step:** H3 heading followed by `<!-- id: step-id -->` or `<!-- id: step-id status: prepared|in-progress|done -->`
- **Full step ID format:** `{phase-id}/{step-id}`

### Status Logic

- No `status` attribute = todo
- `status: prepared` = packet exists, ready to execute
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
- No todo steps for ahead: `"No todo steps remaining to prepare ahead."`
- Ahead mode with step-id: `"--ahead cannot be combined with a specific step-id. Use --ahead to batch-prepare next N todo steps, or specify a step-id for single prepare."`

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

### 2A. Ahead Mode Branch (when `--ahead` is set)

> If `AHEAD_COUNT` is NOT set, skip to Step 3.

If `STEP_ID` is provided → STOP with "Ahead mode with step-id" error.

<ahead-mode>

**Find steps to prepare:**
- Filter steps to those with no status attribute (todo only) — skip `prepared`, `in-progress`, `done`
- Take the first `AHEAD_COUNT` todo steps (in plan order)
- If zero todo steps → STOP with "No todo steps remaining to prepare ahead."
- If fewer todo steps than `AHEAD_COUNT`, use what's available

**Launch parallel Task agents:**

For each todo step, launch a Task agent (`subagent_type: general-purpose`) with this prompt:

```
You are preparing a step packet for an implementation plan.

## Source Documents

Plan file: {PLAN_PATH}
Design document: {DESIGN_DOC_PATH}

## Step to Prepare

Step ID: {phase-id}/{step-id}
Step title: {step heading}

## Instructions

1. Read the plan file and design document thoroughly
2. Extract the step content from the plan (heading + text until next H2/H3)
3. Gather ALL relevant context from the design document and plan
4. Compute packet path: {packet-path}
5. Create the packet markdown file with YAML frontmatter and all sections:
   - Overview, Step Definition, From Design Document, From Plan (Supporting Sections),
     Dependencies, Test Cases, Acceptance Criteria, Implementation Notes
6. Do NOT modify the plan file (status updates are handled by the parent)
7. Do NOT write code or implementation files — ONLY the packet .md file

{If AUTO_CHECK is true, add:}
8. After writing the packet, run a check loop:
   a. Re-read the plan and design doc
   b. Compare against the packet — find ANY missing details
   c. If findings exist → edit the packet to add missing information, then repeat from (a)
   d. If zero findings OR 10 iterations reached → stop
   e. Update the packet's YAML frontmatter confidence fields:
      - If converged (zero findings before 10 iterations): set `check-confidence: converged` and `check-iterations: N`
      - If did not converge (10 iterations reached): set `check-confidence: max-iterations` and `check-iterations: 10`
      - Remove `check-confidence: unchecked` if present
   f. Report: number of check iterations and whether converged (clean) or not

Report back: step ID, packet path, success/failure, check iterations (if auto-check).
```

Launch ALL agents in a single message (parallel Task tool calls).

**After all agents complete:**

For each successful agent:
- Update plan: change `<!-- id: {step-id} -->` to `<!-- id: {step-id} status: prepared -->`

For each failed agent:
- Leave step as todo (no status change)
- Collect error message for report

**Report and STOP:**

```
✅ Prepared {N} steps ahead:

| Step | Packet | Check |
|------|--------|-------|
| {step-id-1} | {packet-path-1} | {converged after M iterations / skipped / failed} |
| {step-id-2} | {packet-path-2} | {converged after M iterations / skipped / failed} |
...

{If any failures:}
⚠️ Failed to prepare: {step-id-x} — {error message}
```

STOP here. Do not continue to normal mode steps.

</ahead-mode>

### 3. Find Step to Prepare (Normal Mode)

<step-selection>
**If STEP_ID is provided:**
- Find the step matching `STEP_ID` (format: `phase-id/step-id`)
- If not found → STOP with "Step not found" error
- If step has `status: done` → STOP with "Step already done" error
- If step has `status: prepared`:
  - Set status to `in-progress` in plan (change `status: prepared` → `status: in-progress`)
  - Compute packet path and verify packet exists
  - If packet exists → Report: "Using pre-generated packet for {step-id}." and skip to Step 11
  - If packet missing → continue to create packet (status is now in-progress)
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
- If none in-progress:
  - Find first non-done step in plan order
  - If that step has `status: prepared`:
    - Set status to `in-progress` in plan (change `status: prepared` → `status: in-progress`)
    - Compute packet path and verify packet exists
    - If packet exists → Report: "Using pre-generated packet for {step-id}." and skip to Step 11
    - If packet missing → continue to create packet (status is now in-progress)
  - If that step has no status (todo) → this is the step to prepare
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

<reading-rules>
**Do NOT use the `limit` parameter** when reading source documents. The Read tool's default behavior reads the full file — that is what you want.

**Anti-pattern (BLOCKED by hook):**
```
Read(file_path: "design.md", limit: 200)  ← WRONG: skips 90%+ of the document
```

**Correct pattern:**
```
Read(file_path: "design.md")              ← RIGHT: reads full file
```

**For very large files (>2000 lines):** use sequential offset reads to cover the entire file:
```
Read(file_path: "design.md")                        ← first 2000 lines
Read(file_path: "design.md", offset: 2000)           ← next chunk
Read(file_path: "design.md", offset: 4000)           ← and so on
```

**Self-check after reading:** Can you recall content from the LAST section of the document? If not, you didn't read far enough.
</reading-rules>

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
check-confidence: unchecked
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

### 10A. Auto-Check Loop (when `--auto-check` is set, normal mode only)

> If `AUTO_CHECK` is false, skip to Step 11.

After packet creation, launch a Task agent (`subagent_type: general-purpose`) for the check loop:

```
You are checking and improving a step packet for completeness.

## Source Documents

Packet: {PACKET_PATH}
Plan file: {PLAN_PATH}
Design document: {DESIGN_DOC_PATH}

## Instructions

Run a check loop until the packet is clean or you reach 10 iterations:

1. Read the packet, plan, and design document thoroughly
2. Compare the packet against source documents — find ANY missing details
   Use this checklist:
   - All technical specifications from design doc relevant to this step
   - Error handling requirements and exact error messages
   - Validation rules, edge cases, type definitions
   - Test scenarios, acceptance criteria completeness
   - Cross-references resolved to actual content
   - Implementation gotchas and warnings
3. Stricter threshold: ANY finding, no matter how small, counts as a gap
4. If findings exist:
   - Edit the packet file to add missing information
   - Do NOT duplicate existing content
   - Do NOT remove existing content
   - Increment iteration counter, go to step 1
5. If zero findings OR iteration counter >= 10 → stop
6. Update the packet's YAML frontmatter confidence fields:
   - If converged (zero findings before 10 iterations): set `check-confidence: converged` and `check-iterations: N`
   - If did not converge (10 iterations reached): set `check-confidence: max-iterations` and `check-iterations: 10`
   - Remove `check-confidence: unchecked` if present

Report: "Check converged after N iterations" or "Check did not converge after 10 iterations — review manually"
```

Wait for the agent to complete and include the check result in the report.

### 11. Report Result

Output:

```
✅ Step prepared: {phase-id}/{step-id}

Title: {step heading}
Packet: {packet-path}
{If auto-check ran: "Check: converged after N iterations" or "Check: did not converge (10 iterations)"}

Next: Run `/ManageImpStep execute {packet-path}` to implement.
```

## Report

After successful preparation, always include:

1. Confirmation message with step ID
2. Full path to the created packet
3. Auto-check result (if `--auto-check` was used)
4. Suggested next command

**Remember:** You prepared a PACKET (documentation). You did NOT implement anything. Implementation is the user's next step via `/ManageImpStep execute`.

## Auto-Suggest

After successful preparation, auto-suggest the next command for the user:

If `--auto-check` was used (packet already checked):
```
/ManageImpStep execute {PACKET_PATH} -n
```

If `--auto-check` was NOT used:
```
/ManageImpStep check {PACKET_PATH} -n
```

Replace `{PACKET_PATH}` with the actual packet path created in this run.
