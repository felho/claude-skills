---
description: Create step implementation packet from plan + design doc
argument-hint: <plan> <design-doc> [phase-id/step-id] [--auto-check]
allowed-tools: Read, Write, Edit, Glob, AskUserQuestion, Skill, Task
# Note: Write/Edit are ONLY for the packet .md file, never for implementation files
# Note: Skill is for invoking Check workflow during auto-check
# Note: Task is for spawning background Check orchestrator during auto-check
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
STEP_ID: \$3 (may be absent)
AUTO_CHECK: from `--auto-check` flag (default: false)

**Flag parsing:** Scan all arguments for `--auto-check` (boolean). Remove flags from positional arguments before processing.

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
- Step done: `"Step {step-id} is already done. To re-implement, remove status: done from the HTML comment, then run /ManageImpStep prepare."`
- All complete: `"All steps are complete (or in-progress by other instances)."`
- Step not found: `"Step not found: {step-id}"`

## Workflow

### 1. Validate Input Files

- If `PLAN_PATH` is empty → STOP and report: `"Usage: /ManageImpStep prepare <plan> <design-doc> [step-id] [--auto-check]"`
- If `DESIGN_DOC_PATH` is empty → STOP and report: `"Usage: /ManageImpStep prepare <plan> <design-doc> [step-id] [--auto-check]"`
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
- If step has `status: in-progress` or `status: prepared`:
  - Compute packet path and check if packet exists
  - If packet exists → Ask user: "Packet already exists. Overwrite?"
    - If user declines → STOP (no changes)
    - If user accepts → continue to create packet
  - If packet doesn't exist → continue to create packet
  - Ensure status is `in-progress` in plan (change if needed)
- Otherwise (todo) → this is the step to prepare
  - **Immediately** set status to `in-progress` in plan (change `<!-- id: {step-id} -->` → `<!-- id: {step-id} status: in-progress -->`)

**If STEP_ID is NOT provided:**

> **⚠️ CRITICAL:** Only `todo` steps (no status attribute) are candidates. You MUST skip ALL steps that have ANY status — `done`, `in-progress`, AND `prepared`. Do NOT treat an `in-progress` step as "the current step to work on" — it was claimed by a previous invocation. Your job is to find the NEXT unclaimed step.
>
> **Anti-pattern trap:** After reading the plan, do NOT investigate `in-progress` or `prepared` steps (check their packets, read their status, glob for related files, etc.). Your ONLY job is to find the first `todo` step. Go directly from parsing → selecting the first todo → announcing it (Step 3A). No detours.

- Scan steps in plan order. Skip every step that has a `status` attribute (`done`, `in-progress`, `prepared`). Select the first step with NO status attribute (todo).
- If found → this is the step to prepare
  - **Immediately** set status to `in-progress` in plan (change `<!-- id: {step-id} -->` → `<!-- id: {step-id} status: in-progress -->`)
  - This claims the step early, preventing parallel instances from selecting the same step
- If no todo steps remain → STOP with message: "All steps are complete (or in-progress by other instances)."
</step-selection>

### 3A. Announce Selected Step (Early Output)

**Immediately after identifying the step**, output a message to the user so they know what's happening:

```
The next step is `{phase-id}/{step-id}`. Let me read the design doc for relevant context.
```

This MUST be output **before** reading the design doc (Step 5). The user should see which step was selected while the heavy reading is in progress.

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

### 7B. Determine Test Strategy

Decide the `test-strategy` frontmatter value based on the step's nature:

| Strategy | When to use | Execute behavior |
|----------|-------------|-----------------|
| `tdd` (default) | Step produces application code in `src/` or `packages/` | Write tests first, red-green cycle |
| `build-verify` | Step is purely config/infrastructure (install deps, create configs, setup tooling) — no `src/` files in deliverables | Skip test creation; verify via build + existing test suite |

**Decision rule:** Look at the Step Definition checkboxes. If ALL deliverables are configuration files (package.json, tsconfig, .gitignore, etc.) or install commands — and NO `src/` or `packages/*/src/` files are created or modified — use `build-verify`. Otherwise use `tdd`.

### 8. Write Packet Markdown File (NOT implementation!)

Create the packet `.md` file with this structure (this is the ONLY file you write):

```markdown
---
step: {phase-id}/{step-id}
plan: {PLAN_PATH}
design-doc: {DESIGN_DOC_PATH}
created: {ISO-8601 timestamp}
check-confidence: unchecked
test-strategy: {tdd|build-verify}
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

### 9. STOP Check (Self-Verification)

Before reporting, verify you followed the rules:

- [ ] You created ONLY one new file: the packet `.md` at the computed path
- [ ] You did NOT create/modify any code, config, or project files
- [ ] You did NOT implement the step (no `special-rules.json`, no source files, etc.)
- [ ] The packet contains context TO BE IMPLEMENTED LATER in Execute workflow

If any check fails → you made a mistake. Undo the implementation work and focus only on the packet.

### 9A. Auto-Check (when `--auto-check` is set)

> If `AUTO_CHECK` is false, skip to Step 10.

After packet creation, spawn a **background Task agent** to run the Check orchestrator:

```
Task(
  description: "Check packet {STEP_ID}",
  subagent_type: "general-purpose",
  run_in_background: true,
  prompt: "Use the Skill tool to invoke: /ManageImpStep check {PACKET_PATH} --orchestrate --max-cycles 3"
)
```

This spawns a background agent that:
- Invokes Check in orchestrator mode (which itself spawns single-pass iterations)
- Runs Phase A (iterative single-pass checks until convergence or 10 iterations)
- Runs Phase B (7-dimension structured double-check)
- Updates packet frontmatter with confidence metadata
- Hooks fire automatically (skill invocation in subagent)

**Do NOT wait for the background agent to complete.** Proceed immediately to Step 10.

Note the `output_file` path from the Task result — include it in the report so the user can check progress later.

### 10. Report Result

Output:

```
✅ Step prepared: {phase-id}/{step-id}

Title: {step heading}
Packet: {packet-path}
{If auto-check was spawned:}
  Check: running in background (output: {output_file})
{If auto-check was NOT spawned:}
  (no check line)

Next: Run `/ManageImpStep check {packet-path}` to verify, then `/ManageImpStep execute {packet-path}` to implement.
```

**Note:** When auto-check is active, the background agent updates the packet frontmatter asynchronously. The user can check progress by reading the output file, or wait and inspect the packet's `check-confidence` frontmatter field later.

## Report

After successful preparation, always include:

1. Confirmation message with step ID
2. Full path to the created packet
3. Background check status with output file path (if `--auto-check` was used)
4. Suggested next command

**Remember:** You prepared a PACKET (documentation). You did NOT implement anything. Implementation is the user's next step via `/ManageImpStep execute`.

## Auto-Suggest

After successful preparation, auto-suggest the next command for the user:

If `--auto-check` was used (check running in background):
```
Check is running in background. When it completes, check the packet's frontmatter for `check-confidence`, then:
/ManageImpStep execute {PACKET_PATH} -n
```

If `--auto-check` was NOT used:
```
/ManageImpStep check {PACKET_PATH} -n
```

Replace `{PACKET_PATH}` with the actual packet path created in this run.

## Parallel Preparation

To prepare multiple steps in parallel, run multiple Claude Code instances simultaneously. Each instance runs `/ManageImpStep prepare` with an explicit `step-id` to avoid conflicts:

```
# Instance 1:
/ManageImpStep prepare plans/myplan.md docs/design.md phase/step-a --auto-check

# Instance 2:
/ManageImpStep prepare plans/myplan.md docs/design.md phase/step-b --auto-check
```

Each instance has the full Skill tool available, so hooks fire correctly. Use explicit `step-id` to avoid two instances picking the same step.
