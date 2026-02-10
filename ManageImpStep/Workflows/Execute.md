---
description: Run implementation based on step packet
argument-hint: <packet>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
hooks:
  Stop:
    - hooks:
        - type: command
          command: "uv run $HOME/.claude/hooks/validators/ManageImpStep/cross-step-consistency-validator.py --mode readiness"
        - type: command
          command: "uv run $HOME/.claude/hooks/validators/ManageImpStep/check-confidence-validator.py"
---

# Execute Step Implementation

Implement a step based on the detailed instructions in its packet. Follow TDD: write tests first, then implementation.

> **🛑 CRITICAL SCOPE BOUNDARY — READ THIS FIRST**
>
> This workflow is **ONLY for implementation**. After completing this workflow, you MUST STOP and wait for the user.
>
> **This workflow DOES:**
>
> - Write tests (TDD)
> - Write implementation code
> - Run tests to verify they pass
> - Type check
>
> **This workflow does NOT (NEVER do these):**
>
> - ❌ Update the plan file (no status changes, no checkbox updates)
> - ❌ Create git commits
> - ❌ Mark the step as done
> - ❌ Invoke the Done workflow
> - ❌ Invoke the UseGit skill
>
> **After this workflow:** User will run `/ManageImpStep validate`, then `/ManageImpStep done`.

> **⚠️ SCOPE: Implement EXACTLY what the packet specifies — no more, no less.**
>
> **Do NOT:**
>
> - Add features beyond what is documented in the packet
> - Refactor code outside the step's scope
> - "Improve" adjacent code while you're there
> - Fix unrelated issues you notice
>
> If you discover something that needs fixing but is outside scope, note it for a future step.

## Variables

PACKET_PATH: $1

## Instructions

### Required YAML Frontmatter Fields

- `step`: Full step ID (phase-id/step-id)
- `plan`: Path to implementation plan
- `design-doc`: Path to design document
- `created`: ISO-8601 timestamp

### TDD Workflow

**Tests first, implementation second.** This is non-negotiable.

1. Read the Test Cases section in the packet
2. Write or update test files BEFORE writing implementation
3. Run tests — they should FAIL (red)
4. Write implementation code
5. Run tests — they should PASS (green)
6. Refactor if needed, keeping tests green

### Implementation Rules

- Implement EXACTLY what the packet specifies — no more, no less
- Do not add features beyond what is documented
- Follow the acceptance criteria strictly
- Use error messages exactly as specified in Implementation Notes
- Reference the Design Document section for technical details

### Error Messages (Use Exactly)

- Packet not found: `"Packet not found: {path}"`
- Invalid frontmatter: `"Invalid packet: missing required frontmatter fields (step, plan, design-doc)"`
- Plan not found: `"Plan file not found: {path}"`
- Step not in-progress (no status): `"Step {step-id} is not in-progress (status may have been manually removed). Run /ManageImpStep prepare to resume."`
- Step already done: `"Step {step-id} is already done. Remove status: done from the HTML comment, then run /ManageImpStep prepare."`

## Workflow

### 1. Validate Input

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep execute <packet>"`
- Read packet file at `PACKET_PATH`
  - If file not found → STOP with "Packet not found" error

### 2. Parse Packet Frontmatter

Extract from YAML frontmatter:

- `step` → STEP_ID
- `plan` → PLAN_PATH
- `design-doc` → DESIGN_DOC_PATH

If any required field is missing → STOP with "Invalid packet" error

### 3. Verify Step Status in Plan

- Read plan file at `PLAN_PATH`
  - If file not found → STOP with "Plan file not found" error
- Find the step's HTML comment: `<!-- id: {step-id} ... -->`
- Check status:
  - If no `status` attribute → STOP with "Step not in-progress" error
  - If `status: done` → STOP with "Step already done" error
  - If `status: in-progress` → continue

### 4. Read Packet Content

Read the full packet and extract:

<packet-sections>
- **Step Definition** — What to build (checkboxes, tasks)
- **From Design Document** — Technical specifications
- **From Plan (Supporting Sections)** — Additional context
- **Dependencies** — Prerequisites (verify they are complete)
- **Test Cases** — What to test
- **Acceptance Criteria** — Definition of done
- **Implementation Notes** — Gotchas, exact error messages
</packet-sections>

### 4b. Explore Codebase for Context

Before writing tests, gather context about the existing codebase:

- Use the **Task tool with `subagent_type=Explore`** to find existing patterns, related files, and dependencies
- **Do NOT specify `allowed_tools`** on the Task call — Explore agents already have Read/Glob/Grep access by default. Specifying `allowed_tools` triggers an unnecessary permission prompt for the user.

### 5. Write Tests First

Based on Test Cases section:

<write-tests>
1. Identify test file location (create if needed)
2. Write test cases covering:
   - Happy path scenarios
   - Edge cases from Implementation Notes
   - Error conditions
3. Run tests to confirm they FAIL (red phase)
   - Use `bun run test` or project's test command
   - Tests should fail because implementation doesn't exist yet
</write-tests>

### 6. Implement the Step

Based on Step Definition and Design Document sections:

> **⚠️ Pre-Edit Checklist — EVERY file, EVERY time**
>
> Before editing ANY existing `.ts` file, run the Pre-Edit Checklist:
>
> ```
> Pre-Edit Checklist for [filename.ts]:
> - [ ] Read the test file: [test filename or "no test file exists"]
> - [ ] Tests need updating: yes/no/no test file
> - [ ] If yes: edit test FIRST, run tests, then edit implementation
> ```
>
> **The refactoring trap:** When refactoring existing code to use a new shared helper,
> it's tempting to treat it as "just moving code around." But if behavior changes
> (e.g., checking full prerequisite chain instead of just immediate predecessor),
> the existing tests must be updated FIRST to cover the new behavior.
>
> See: [PreEditCheck workflow](../../WriteCode/Workflows/PreEditCheck.md)

<implement>
1. Create/modify files as specified in Step Definition
2. **For each existing file:** Run Pre-Edit Checklist before editing
3. Follow patterns from Design Document
4. Use exact error messages from Implementation Notes
5. Handle edge cases documented in the packet
6. Keep implementation minimal — only what's specified
</implement>

### 7. Run Tests

- Run the test suite: `bun run test` (or project's test command)
- All tests should PASS (green phase)
- If tests fail:
  - Fix the implementation
  - Re-run tests
  - Repeat until green

### 8. Verify Against Acceptance Criteria

Go through each acceptance criterion in the packet:

<verify-criteria>
- [ ] Criterion 1 — manually verify or check test coverage
- [ ] Criterion 2 — verify
- [ ] ... continue for all criteria
</verify-criteria>

If any criterion is not met → continue implementation until all pass

### 9. Type Check

Run TypeScript type check (if applicable):

```bash
tsc --noEmit
```

Fix any type errors before completing.

### 10. STOP Check (Scope Verification)

Before reporting, verify you stayed in scope:

- [ ] You ONLY implemented what was specified in the packet
- [ ] You did NOT add extra features, helpers, or "improvements"
- [ ] You did NOT refactor code outside the step's scope
- [ ] You did NOT fix unrelated issues (note them for future steps instead)

If any check fails → undo the out-of-scope work before proceeding.

## Report

After implementation, provide:

```
✅ Step implemented: {STEP_ID}

Files created/modified:
- {file1}
- {file2}

Tests: {passed}/{total} passing

Next: Run `/ManageImpStep validate {PACKET_PATH}` to verify acceptance criteria.
```

**If implementation is incomplete or blocked:**

```
⚠️ Step partially implemented: {STEP_ID}

Completed:
- {what was done}

Blocked by:
- {reason}

Next steps:
- {what needs to happen}
```

## Auto-Suggest

After successful implementation (all tests pass), auto-suggest the next command for the user:

```
/ManageImpStep validate {PACKET_PATH} -n
```

Replace `{PACKET_PATH}` with the actual packet path used in this run.

---

## 🛑 HARD STOP — DO NOT PROCEED

**This workflow is complete. You MUST stop here.**

Do NOT:

- Update the plan file
- Create a git commit
- Mark checkboxes as done
- Change step status to "done"
- Invoke `/ManageImpStep done`
- Invoke the UseGit skill

**Wait for the user** to run `/ManageImpStep validate` and then `/ManageImpStep done`.

If you find yourself about to do any of the above actions, STOP immediately. These actions belong to the Done workflow, not Execute.
