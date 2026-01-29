---
description: Apply targeted fixes for failed validation criteria using findings file
argument-hint: <packet-path> [extra-feedback]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Fix Failed Validation Criteria

Apply targeted TDD fixes for criteria that failed validation. Reads the findings file produced by `/ManageImpStep validate` and fixes ONLY the identified issues.

> **🛑 CRITICAL SCOPE BOUNDARY — READ THIS FIRST**
>
> This workflow is **ONLY for fixing issues identified in the findings file**. After completing this workflow, you MUST STOP and wait for the user.
>
> **This workflow DOES:**
>
> - Read the findings file to understand what failed
> - Read relevant packet sections for the failed criteria
> - Update tests for the failed criteria (TDD)
> - Fix implementation for the failed criteria
> - Run tests to verify fixes
> - Type check
>
> **This workflow does NOT (NEVER do these):**
>
> - ❌ Fix issues NOT listed in the findings file
> - ❌ Refactor or improve passing code
> - ❌ Add features beyond what the findings require
> - ❌ Update the plan file (no status changes, no checkbox updates)
> - ❌ Create git commits
> - ❌ Mark the step as done
> - ❌ Run validation (user runs `/ManageImpStep validate` after this)
> - ❌ Invoke the Done workflow or UseGit skill
>
> **After this workflow:** User will run `/ManageImpStep validate` to re-check.

## Variables

PACKET_PATH: $1
EXTRA_FEEDBACK: $2 (optional, additional context from user beyond findings file)
FINDINGS_PATH: PACKET_PATH with `.md` replaced by `.findings.md`
Example: `decide-add-mapping.md` → `decide-add-mapping.findings.md`

## Instructions

### Required YAML Frontmatter Fields (in packet)

- `step`: Full step ID (phase-id/step-id)
- `plan`: Path to implementation plan
- `design-doc`: Path to design document
- `created`: ISO-8601 timestamp

### Fix Philosophy

**Targeted correction, not re-implementation.** This workflow fixes ONLY what the validation identified as failing. Every change must trace back to a specific failed criterion in the findings file.

### TDD for Fixes

**Tests first, fix second.** For each failed criterion:

1. Write or update test(s) that cover the missing behavior
2. Run tests — new test(s) should FAIL (red)
3. Implement the fix
4. Run tests — all should PASS (green)

### Error Messages (Use Exactly)

- Packet not found: `"Packet not found: {path}"`
- Invalid frontmatter: `"Invalid packet: missing required frontmatter fields (step, plan, design-doc)"`
- Plan not found: `"Plan file not found: {path}"`
- Step not in-progress (no status): `"Step {step-id} is not in-progress (status may have been manually removed). Run /ManageImpStep prepare to resume."`
- Step already done: `"Step {step-id} is already done. Remove status: done from the HTML comment, then run /ManageImpStep prepare."`
- No findings file: `"No findings file found at {FINDINGS_PATH}. Run /ManageImpStep validate {PACKET_PATH} first."`
- Findings show pass: `"Findings show PASS — nothing to fix. Run /ManageImpStep done {PACKET_PATH} to complete the step."`

## Workflow

### 1. Validate Input

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep fix <packet> [extra-feedback]"`
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

### 4. Read Findings File

- Read file at `FINDINGS_PATH`
  - If file not found → STOP with "No findings file" error
- Parse YAML frontmatter:
  - If `result: PASS` → STOP with "Findings show pass" error
  - If `result: FAIL`, `PARTIAL`, or `BLOCKED` → continue
- Extract the `## Failed` section → list of failed criteria

### 5. Read Relevant Packet Sections

For each failed criterion, use its `**Packet section**` field to identify which part of the packet to read. Read ONLY those sections — do not load the entire packet.

<read-packet-sections>
- Read the referenced section from the packet
- Understand the expected behavior and specification
- Note any related test cases or implementation notes
</read-packet-sections>

If `EXTRA_FEEDBACK` is provided, incorporate it as additional context for the fix.

### 6. Fix Each Failed Criterion (TDD)

For each failed criterion in the findings:

<fix-criterion-loop>

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

1. **Write/update test(s)** for the specific failed criterion
2. **Run tests** — new test(s) should FAIL (confirms the gap exists)
3. **Implement the fix** — modify only what's needed for this criterion
4. **Run tests** — all tests should PASS (confirms the fix works)

</fix-criterion-loop>

### 7. Run Full Test Suite

After all fixes are applied:

```bash
bun run test
```

(Or project's test command if different)

- All tests must PASS — both new and existing
- If any test fails → fix before proceeding

### 8. Type Check

Run TypeScript type check (if applicable):

```bash
tsc --noEmit
```

Fix any type errors before completing.

### 9. STOP Check (Scope Verification)

Before reporting, verify you stayed in scope:

- [ ] You ONLY fixed criteria listed in the findings file
- [ ] You did NOT refactor or improve passing code
- [ ] You did NOT add features beyond what the findings require
- [ ] You did NOT fix unrelated issues
- [ ] Every change traces back to a specific failed criterion

If any check fails → undo the out-of-scope work before proceeding.

## Report

After fixing, provide:

```
✅ Fixed {N} issue(s) for: {STEP_ID}

Changes:
- {criterion number}. {brief description of fix}
- {criterion number}. {brief description of fix}

Tests: {passed}/{total} passing

Next: Run `/ManageImpStep validate {PACKET_PATH}` to verify.
```

**If fix is incomplete or blocked:**

```
⚠️ Partially fixed: {STEP_ID}

Fixed:
- {criterion number}. {what was fixed}

Could not fix:
- {criterion number}. {reason}

Next steps:
- {what needs to happen}
```

---

## 🛑 HARD STOP — DO NOT PROCEED

**This workflow is complete. You MUST stop here.**

Do NOT:

- Run `/ManageImpStep validate`
- Update the plan file
- Create a git commit
- Mark checkboxes as done
- Change step status to "done"
- Invoke `/ManageImpStep done`
- Invoke the UseGit skill

**Wait for the user** to run `/ManageImpStep validate` to re-check the fixes.

If you find yourself about to do any of the above actions, STOP immediately. These actions belong to other workflows, not Fix.
