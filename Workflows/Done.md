---
description: Mark step complete in plan file, check all checkboxes, set status to done
argument-hint: <packet> [--commit]
allowed-tools: Read, Edit, Bash, Glob, Grep, Skill
---

# Mark Step Done

Mark a step as complete in the implementation plan. Updates the step's HTML comment status to `done` and checks all task checkboxes within the step.

## Variables

PACKET_PATH: $1
COMMIT_FLAG: $2 (optional, "--commit" to create git commit after marking done)

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

### Commit Behavior

When `--commit` flag is provided:
- Follow UseGit skill practices
- Create conventional commit message: `feat(plan): complete step {step-id}`
- Include modified plan file in commit
- Do NOT re-run validation — user's responsibility to validate first

### Error Messages (Use Exactly)

- Packet not found: `"Packet not found: {path}"`
- Invalid frontmatter: `"Invalid packet: missing required frontmatter fields (step, plan, design-doc)"`
- Plan not found: `"Plan file not found: {path}"`
- Step not in-progress (no status): `"Step {step-id} is not in-progress. Run /ManageImpStep prepare first."`
- Step already done: `"Step {step-id} is already done. To re-implement, remove status: done from the HTML comment, then run /ManageImpStep prepare."`

## Workflow

### 1. Validate Input

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep done <packet> [--commit]"`
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

### 9. Handle Commit (if requested)

If `COMMIT_FLAG` is `--commit`:

<commit-flow>
1. Invoke UseGit skill for proper commit handling
2. Stage the modified plan file
3. Commit with message: `feat(plan): complete step {STEP_ID}`
4. Include standard Co-Authored-By line
</commit-flow>

If `--commit` not provided, skip this step.

## Report

### Success (without commit):

```
✅ Step marked done: {STEP_ID}

Plan updated: {PLAN_PATH}
- Status: in-progress → done
- Checkboxes: {checked_count} checked

Next step: Run `/ManageImpStep prepare {PLAN_PATH} {DESIGN_DOC_PATH}` to prepare the next step.
```

### Success (with commit):

```
✅ Step marked done: {STEP_ID}

Plan updated: {PLAN_PATH}
- Status: in-progress → done
- Checkboxes: {checked_count} checked

Commit created: feat(plan): complete step {STEP_ID}

Next step: Run `/ManageImpStep prepare {PLAN_PATH} {DESIGN_DOC_PATH}` to prepare the next step.
```

### Error:

```
❌ {error_message}
```
