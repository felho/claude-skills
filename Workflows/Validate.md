---
description: Verify implementation meets acceptance criteria from packet
argument-hint: <packet>
allowed-tools: Read, Bash, Glob, Grep
---

# Validate Step Implementation

Verify that the implementation meets ALL acceptance criteria defined in the step packet. Run automated checks and report pass/fail status.

## Variables

PACKET_PATH: $1

## Instructions

### Required YAML Frontmatter Fields

- `step`: Full step ID (phase-id/step-id)
- `plan`: Path to implementation plan
- `design-doc`: Path to design document
- `created`: ISO-8601 timestamp

### Validation Philosophy

**Thorough verification, not re-implementation.** This workflow checks that what was built meets the specification — it does not fix issues or write code.

### Check Categories

1. **Type Check** — TypeScript compiles without errors
2. **Test Suite** — All tests pass
3. **File Existence** — Required files exist
4. **Acceptance Criteria** — Each criterion from packet is verified

### Error Messages (Use Exactly)

- Packet not found: `"Packet not found: {path}"`
- Invalid frontmatter: `"Invalid packet: missing required frontmatter fields (step, plan, design-doc)"`
- Plan not found: `"Plan file not found: {path}"`
- Step not in-progress (no status): `"Step {step-id} is not in-progress (status may have been manually removed). Run /ManageImpStep prepare to resume."`
- Step already done: `"Step {step-id} is already done. Remove status: done from the HTML comment, then run /ManageImpStep prepare."`

## Workflow

### 1. Validate Input

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep validate <packet>"`
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

### 4. Extract Validation Criteria from Packet

Read the full packet and extract:

<validation-sections>
- **Step Definition** — Files to create, checkboxes (implies file existence checks)
- **Test Cases** — Tests that should pass
- **Acceptance Criteria** — Definition of done (each criterion to verify)
- **Implementation Notes** — Any specific validation requirements
</validation-sections>

### 5. Run Type Check

```bash
tsc --noEmit
```

- Record result: PASS or FAIL with error details
- If project doesn't use TypeScript, skip this check

### 6. Run Test Suite

```bash
bun run test
```

(Or project's test command if different)

- Record result: PASS or FAIL with failing test details
- Count: passed/total tests

### 7. Verify File Existence

For each file mentioned in Step Definition:

<file-check>
- Check if file exists using Glob or Read
- Record: exists or missing
</file-check>

### 8. Verify Acceptance Criteria

For each criterion in the Acceptance Criteria section:

<criteria-check>
- Determine verification method:
  - Code behavior → check via test results
  - File content → read and verify
  - Configuration → inspect file
  - Manual check → note as "requires manual verification"
- Record: PASS, FAIL, or MANUAL
</criteria-check>

### 9. Compile Results

Aggregate all check results:
- Type check: PASS/FAIL
- Tests: X/Y passing
- Files: all exist / N missing
- Acceptance criteria: X/Y verified, Z manual

Determine overall status:
- **PASS** — All automated checks pass
- **FAIL** — Any automated check fails
- **PARTIAL** — Automated pass, but has manual verification items

## Report

### If ALL checks PASS:

```
✅ Validation PASSED: {STEP_ID}

Type check: ✓ No errors
Tests: {passed}/{total} passing
Files: All required files exist
Acceptance criteria: {verified}/{total} verified

Next: Run `/ManageImpStep done {PACKET_PATH}` to mark step complete.
```

### If ANY check FAILS:

```
❌ Validation FAILED: {STEP_ID}

Type check: {✓ or ✗ with error count}
Tests: {passed}/{total} passing
  - {failing test 1}
  - {failing test 2}
Files: {status}
  - Missing: {file1}, {file2}
Acceptance criteria: {verified}/{total}
  - ✗ {failed criterion}

Fix the issues and re-run `/ManageImpStep validate {PACKET_PATH}`.
```

### If PARTIAL (has manual checks):

```
⚠️ Validation PARTIAL: {STEP_ID}

Automated checks: ✓ All passed
- Type check: ✓
- Tests: {passed}/{total}
- Files: ✓

Manual verification needed:
- [ ] {criterion requiring manual check}
- [ ] {another manual criterion}

If manual checks pass, run `/ManageImpStep done {PACKET_PATH}`.
```
