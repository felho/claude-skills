---
description: Mark step complete in plan file, check all checkboxes, set status to done
argument-hint: <packet> [--no-commit]
allowed-tools: Read, Edit, Bash, Glob, Grep, Skill
# Note: Edit is ONLY for updating the plan .md file, never for implementation files
---

# Mark Step Done

Mark a step as complete in the implementation plan. Updates the step's HTML comment status to `done` and checks all task checkboxes within the step.

> **⚠️ SCOPE: This workflow ONLY marks completion — no implementation work.**
>
> **Do NOT:**
>
> - Do "one more fix" before marking done
> - Add finishing touches to the code
> - Refactor or improve implementation
> - Create or modify implementation files
>
> If the step isn't ready to be marked done, use `/ManageImpStep execute` or `/ManageImpStep validate` instead.

## Variables

PACKET_PATH: $1
NO_COMMIT_FLAG: $2 (optional, "--no-commit" to skip git commit)

## Instructions

### Required YAML Frontmatter Fields

- `step`: Full step ID (phase-id/step-id)
- `plan`: Path to implementation plan
- `design-doc`: Path to design document
- `created`: ISO-8601 timestamp

### Checkbox Handling

- Only check unchecked boxes (`- [ ]` → `- [x]`)
- Already checked boxes remain unchanged
- Steps without checkboxes are valid — only status attribute is updated

### Status Update Pattern

Change `<!-- id: {step-id} status: in-progress -->` to `<!-- id: {step-id} status: done -->`

### Commit Behavior (Default)

By default, Done creates a git commit with all step-related changes. Use `--no-commit` to skip.

- Follow UseGit skill practices
- **Commit ALL step-related changes:** implementation files, plan update, packet file
- Create conventional commit message: `feat({scope}): complete step {step-id}`
  - Use the project/app name as scope (e.g., `feat(personal-finance): complete step ...`)
- Do NOT re-run validation — user's responsibility to validate first

> **Why commit by default?** The Done workflow is the final step after Execute and Validate. This is when all implementation is complete and verified, making it the natural place to commit everything together.

### Error Messages (Use Exactly)

- Packet not found: `"Packet not found: {path}"`
- Invalid frontmatter: `"Invalid packet: missing required frontmatter fields (step, plan, design-doc)"`
- Plan not found: `"Plan file not found: {path}"`
- Step not in-progress (no status): `"Step {step-id} is not in-progress. Run /ManageImpStep prepare first."`
- Step already done: `"Step {step-id} is already done. To re-implement, remove status: done from the HTML comment, then run /ManageImpStep prepare."`

## Workflow

### 1. Validate Input

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep done <packet> [--no-commit]"`
- Read packet file at `PACKET_PATH`
  - If file not found → STOP with "Packet not found" error

### 2. Parse Packet Frontmatter

Extract from YAML frontmatter:

- `step` → STEP_ID (format: phase-id/step-id)
- `plan` → PLAN_PATH

If `step` or `plan` field is missing → STOP with "Invalid packet" error

### 3. Read Plan and Find Step

- Read plan file at `PLAN_PATH`
  - If file not found → STOP with "Plan file not found" error
- Find the step's HTML comment: `<!-- id: {step-id} ... -->`
  - Extract just the step-id part (after the `/` in STEP_ID)

### 4. Verify Step Status

Check the HTML comment for status attribute:

- If no `status` attribute present → STOP with "Step not in-progress" error
- If `status: done` → STOP with "Step already done" error
- If `status: in-progress` → continue

### 5. Find Step Boundaries

Locate the step's content in the plan:

<step-boundary-detection>
- Step starts at the H3 heading line (### Step ...)
- Step ends at:
  - Next H2 heading (`## `), OR
  - Next H3 heading (`### `), OR
  - End of file
</step-boundary-detection>

### 6. Check All Checkboxes Within Step

Within the step boundaries, find all unchecked checkboxes:

<checkbox-update>
- Pattern to find: `- [ ]` (unchecked)
- Replace with: `- [x]` (checked)
- Already checked `- [x]` boxes remain unchanged
</checkbox-update>

### 7. Update Status Attribute

Change the HTML comment from in-progress to done:

<status-update>
- Find: `<!-- id: {step-id} status: in-progress -->`
- Replace: `<!-- id: {step-id} status: done -->`
</status-update>

### 8. Save Plan Changes

Write the modified plan back to `PLAN_PATH` using Edit tool.

### 9. Handle Commit (default)

Unless `--no-commit` flag is provided:

<commit-flow>
1. Invoke UseGit skill for proper commit handling
2. Stage ALL step-related files:
   - Implementation files (new/modified code)
   - Plan file (with updated status and checkboxes)
   - Packet file (step definition)
   - Any other related files (lock files, configs)
3. Commit with message: `feat({scope}): complete step {STEP_ID}`
4. Include standard Co-Authored-By line
</commit-flow>

If `--no-commit` provided, skip this step.

## Report

### Success (default, with commit):

```
✅ Step marked done: {STEP_ID}

Plan updated: {PLAN_PATH}
- Status: in-progress → done
- Checkboxes: {checked_count} checked

Commit created: feat({scope}): complete step {STEP_ID}

Next step: Run `/ManageImpStep prepare {PLAN_PATH} {DESIGN_DOC_PATH}` to prepare the next step.
```

### Success (with --no-commit):

```
✅ Step marked done: {STEP_ID}

Plan updated: {PLAN_PATH}
- Status: in-progress → done
- Checkboxes: {checked_count} checked

Next step: Run `/ManageImpStep prepare {PLAN_PATH} {DESIGN_DOC_PATH}` to prepare the next step.
```

### Error:

```
❌ {error_message}
```
