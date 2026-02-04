---
description: Verify packet completeness against source documents, improve if needed
argument-hint: <packet>
allowed-tools: Read, Edit, Glob
# Note: Edit is ONLY for updating the packet .md file, never for implementation files
---

# Check Step Packet

Verify that a step packet contains all information needed for successful implementation by re-reading source documents and finding missing details.

> **⚠️ SCOPE: This workflow ONLY modifies the packet `.md` file.**
>
> You may ADD missing information to the packet. You must NOT:
>
> - Create or modify implementation files
> - Write code or config files
> - Fix issues in the codebase
>
> If you find the packet incomplete, enhance the packet documentation only.

## Variables

PACKET_PATH: $1

## Instructions

### Required YAML Frontmatter Fields

- `step`: Full step ID (phase-id/step-id)
- `plan`: Path to implementation plan
- `design-doc`: Path to design document
- `created`: ISO-8601 timestamp

### Error Messages (Use Exactly)

- Packet not found: `"Packet not found: {path}"`
- Invalid frontmatter: `"Invalid packet: missing required frontmatter fields (step, plan, design-doc)"`
- Plan not found: `"Plan file not found: {path}"`
- Design doc not found: `"Design document not found: {path}"`
- Step not in-progress or prepared (no status): `"Step {step-id} is not in-progress or prepared. Run /ManageImpStep prepare to resume."`
- Step already done: `"Step {step-id} is already done. Remove status: done from the HTML comment, then run /ManageImpStep prepare."`

## Workflow

### 1. Validate Input

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep check <packet>"`
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
  - If `status: prepared` → continue (packet exists from prepare-ahead)

### 4. Read Source Documents Thoroughly

Read the ENTIRE design document at `DESIGN_DOC_PATH`:

- If file not found → STOP with "Design document not found" error

Read the ENTIRE plan file at `PLAN_PATH`.

**Take your time. Read carefully. Don't skim.**

### 5. Analyze Packet for Missing Information

**Think very hard.** Compare the packet against source documents. Find EVERYTHING that's missing.

<completeness-checklist>
- [ ] All technical specifications from design doc relevant to this step
- [ ] Error handling requirements and exact error messages
- [ ] Validation rules and constraints
- [ ] Edge cases and corner cases
- [ ] Type definitions and interfaces
- [ ] Configuration requirements
- [ ] Dependencies on other steps or components
- [ ] Test scenarios (unit, integration, edge cases)
- [ ] Acceptance criteria completeness
- [ ] Cross-references to other sections (resolved to actual content)
- [ ] Implementation gotchas and warnings
- [ ] Performance considerations if applicable
- [ ] Security considerations if applicable
</completeness-checklist>

**Key question:** If someone implemented this step using ONLY the packet, would they have ALL the information needed? If not, what's missing?

**Stricter threshold:** Any finding, no matter how small, counts as a gap. Do not dismiss findings as "minor" or "not worth fixing." The only valid clean result is zero findings. If a detail is in the source documents but not in the packet, it must be added.

### 6. Capture Potential Issues (Interim List)

Create a numbered list of potential issues found during analysis. This is an interim working list — items may turn out to be already covered.

**Format:**

```
**Potential Issues Found:**
1. [Description of what might be missing]
2. [Another potential gap]
3. [Edge case that may not be covered]
```

**Important:** This list captures your initial findings. Each item will be verified in the next step.

### 7. Verify Each Issue Against Packet

For each potential issue, search the packet for evidence. Provide explicit proof (line numbers, exact quotes) for your verification.

**Format:**

```
**Verification:**
1. [Issue description]
   → ✅ **Found** at line X: "[exact quote from packet]"

2. [Issue description]
   → ✅ **Found** at line Y: "[exact quote]"

3. [Issue description]
   → ❌ **Missing** — needs to be added to packet
```

**Rules:**

- Every potential issue MUST be verified with proof
- Use exact line numbers from the packet
- Quote the relevant text that proves coverage
- Mark clearly: ✅ Found (with proof) or ❌ Missing
- The final verdict must be consistent with verification results

### 8. Update Packet with Missing Information

For each item marked ❌ Missing in verification:

<update-loop>
- Add missing design doc content to "From Design Document" section
- Add missing plan content to "From Plan (Supporting Sections)" section
- Add missing test cases to "Test Cases" section
- Add missing acceptance criteria to "Acceptance Criteria" section
- Add missing gotchas/warnings to "Implementation Notes" section
- Do NOT duplicate information already in the packet
- Do NOT remove existing content
- Use the packet's existing section structure
</update-loop>

### 9. Report Result

**If no changes needed (all issues verified as present):**

```
**Potential Issues Found:**
1. {brief description}
2. {brief description}

**Verification:**
1. {description} → ✅ Found at line X: "{quote}"
2. {description} → ✅ Found at line Y: "{quote}"

---

✅ **Packet is complete:** {PACKET_PATH}

All potential issues verified as present in packet. Ready for implementation.

Next: Run `/ManageImpStep execute {PACKET_PATH}` to implement.
```

**If packet was updated (some issues were missing):**

```
**Potential Issues Found:**
1. {brief description}
2. {brief description}
3. {brief description}

**Verification:**
1. {description} → ✅ Found at line X: "{quote}"
2. {description} → ❌ Missing
3. {description} → ❌ Missing

---

✅ **Packet updated:** {PACKET_PATH}

Added:
- {summary of what was added for issue #2}
- {summary of what was added for issue #3}

Next: Review changes, then run `/ManageImpStep execute {PACKET_PATH}` to implement.
```

## Report

After execution, always include:

1. Status (complete or updated)
2. If updated: summary of additions
3. Suggested next command

## Auto-Suggest

After a successful check (whether packet was already complete or was updated), auto-suggest the next command for the user:

```
/ManageImpStep execute {PACKET_PATH} -n
```

Replace `{PACKET_PATH}` with the actual packet path used in this run.

## Idempotency Note

This workflow is idempotent — running it multiple times will not duplicate information. Each run:

1. Re-reads source documents
2. Checks what's missing compared to current packet state
3. Only adds what's not already present
