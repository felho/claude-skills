---
description: Verify packet completeness against source documents, improve if needed
argument-hint: <packet> [--loop] [--double-check] [--max-cycles N]
allowed-tools: Read, Edit, Glob, Bash
# Note: Edit is ONLY for updating the packet .md file, never for implementation files
# Note: Bash is ONLY for running validator scripts explicitly
hooks:
  PreToolUse:
    - matcher: "Read"
      hooks:
        - type: command
          command: "uv run $HOME/.claude/hooks/validators/ManageImpStep/design-doc-depth-guard.py"
  PostToolUse:
    - matcher: "Edit"
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
LOOP_MODE: from `--loop` flag (default: false)
DOUBLE_CHECK_MODE: from `--double-check` flag (default: false)
MAX_CYCLES: from `--max-cycles N` (default: 3, only meaningful with `--double-check`)

**Flag parsing:** Scan all arguments for `--loop` (boolean), `--double-check` (boolean), and `--max-cycles N` (integer). Remove flags from positional arguments before processing.

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

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep check <packet> [--loop] [--double-check] [--max-cycles N]"`
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

Read the design document at `DESIGN_DOC_PATH`:

- If file not found → STOP with "Design document not found" error

Read the plan file at `PLAN_PATH`.

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

After each packet Edit, run the structure validator explicitly:

```bash
uv run $HOME/.claude/hooks/validators/ManageImpStep/packet-structure-validator.py
```

### 8A. Loop Mode (when `--loop` is set)

> If `LOOP_MODE` is false, skip to Step 8B.

After completing Steps 5-8 once, check if any items were marked ❌ Missing and fixed.

<loop-mode>
If items were fixed, **repeat Steps 4-8**:

1. Re-read source documents (Step 4) — context may have shifted after edits
2. Re-analyze packet for remaining gaps (Steps 5-7)
3. Fix any newly found issues (Step 8)
4. Repeat until zero findings OR 10 iterations reached

**Track:** `iteration-count` starting at 1 (the initial pass counts as iteration 1).

**Convergence:** If an iteration finds zero issues (all verified as ✅ Found), the loop has converged.

**Max iterations:** If 10 iterations reached without convergence, stop the loop.

At the end of Phase A (loop), run coverage and consistency validators:

```bash
uv run $HOME/.claude/hooks/validators/ManageImpStep/packet-coverage-validator.py
uv run $HOME/.claude/hooks/validators/ManageImpStep/cross-step-consistency-validator.py
```
</loop-mode>

### 8B. Double-Check Mode (when `--double-check` is set)

> If `DOUBLE_CHECK_MODE` is false, skip to Step 9.
> Requires `--loop` to also be set (double-check without loop is not supported — if `--double-check` is set without `--loop`, treat as if `--loop` is also set).

After Phase A (loop) converges, run Phase B: structured 7-dimension evaluation.

<double-check-mode>
**Track across cycles:**
- `total-iterations`: cumulative check iterations across all Phase A runs
- `double-check-restarts`: how many times Phase B triggered a restart (0 to MAX_CYCLES-1)

**Phase B — Structured Double-Check (7 dimensions):**

Read the 7 semantic dimensions from `~/.claude/skills/ManageImpStep/references/DoubleCheckPrompt.md`.

For each dimension, evaluate the packet:
1. **Technical Sufficiency** — types, interfaces, configs, file paths all specified?
2. **Error Handling Coverage** — all error cases with messages and recovery paths?
3. **Edge Cases & Validation** — boundary conditions, validation rules, concurrency?
4. **Testability** — test cases concrete enough to write actual test code?
5. **Dependency Clarity** — inter-step deps and external deps explicit?
6. **Acceptance Criteria Clarity** — each AC specific, measurable, binary (pass/fail)?
7. **Implementation Completeness** — could implementer start with ONLY this packet?

For each dimension:
- Quote specific evidence from the packet
- Cross-reference against plan and design doc
- Score: **PASS** / **WEAK** / **MISSING**

**Report Phase B results in table format:**

```
| # | Dimension | Score | Evidence / Gap |
|---|-----------|-------|----------------|
| 1 | Technical Sufficiency | PASS/WEAK/MISSING | [quote or gap] |
| 2 | Error Handling Coverage | PASS/WEAK/MISSING | [quote or gap] |
| 3 | Edge Cases & Validation | PASS/WEAK/MISSING | [quote or gap] |
| 4 | Testability | PASS/WEAK/MISSING | [quote or gap] |
| 5 | Dependency Clarity | PASS/WEAK/MISSING | [quote or gap] |
| 6 | Acceptance Criteria Clarity | PASS/WEAK/MISSING | [quote or gap] |
| 7 | Implementation Completeness | PASS/WEAK/MISSING | [quote or gap] |
```

**If ALL 7 PASS:**
- Phase B is complete. Set confidence to `double-checked`.
- STOP the cycle loop.

**If ANY WEAK or MISSING:**
- Fix the packet (add missing content from plan and design doc).
- Do NOT remove or duplicate existing content.
- Increment `double-check-restarts`.
- If `double-check-restarts` < MAX_CYCLES - 1:
  - RESTART: go back to Phase A (Step 8A loop) to re-verify consistency after fixes.
  - Then run Phase B again.
- If MAX_CYCLES cycles exhausted:
  - Set confidence to `max-double-checks`.
  - STOP.

**If Phase A did not converge (max-iterations):**
- Skip Phase B entirely.
- Set confidence to `max-iterations`.
- STOP.
</double-check-mode>

### 9. Update Frontmatter with Confidence Metadata

Update the packet's YAML frontmatter based on the mode used:

**No flags (single-pass):** Do not modify confidence fields. Leave existing `check-confidence` as-is.

**With `--loop` (Phase A only):**
- If converged: set `check-confidence: converged`, `check-iterations: {N}`
- If max iterations: set `check-confidence: max-iterations`, `check-iterations: 10`

**With `--loop --double-check` (Phase A + Phase B):**
- If all 7 pass: set `check-confidence: double-checked`, `check-iterations: {total}`, `double-check-restarts: {R}`
- If max double-checks: set `check-confidence: max-double-checks`, `check-iterations: {total}`, `double-check-restarts: {MAX_CYCLES - 1}`
- If max iterations (Phase A never converged): set `check-confidence: max-iterations`, `check-iterations: {total}`

Remove `check-confidence: unchecked` if present (replace with actual result).

### 10. Report Result

**Single-pass mode (no flags) — no changes needed:**

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

**Single-pass mode (no flags) — packet was updated:**

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

**With `--loop` (Phase A only):**

```
✅ **Packet checked:** {PACKET_PATH}

Check: converged after {N} iterations
Confidence: converged

Next: Run `/ManageImpStep execute {PACKET_PATH}` to implement.
```

Or if max iterations:

```
⚠️ **Packet checked:** {PACKET_PATH}

Check: did not converge after 10 iterations
Confidence: max-iterations — manual review recommended

Next: Run `/ManageImpStep check {PACKET_PATH}` for manual review, then `/ManageImpStep execute {PACKET_PATH}`.
```

**With `--loop --double-check` (Phase A + Phase B):**

```
✅ **Packet checked:** {PACKET_PATH}

Check: double-checked ✅ ({N} total iterations, {R} restarts)
Confidence: double-checked

| # | Dimension | Score |
|---|-----------|-------|
| 1 | Technical Sufficiency | PASS |
| 2 | Error Handling Coverage | PASS |
| ... | ... | ... |

Next: Run `/ManageImpStep execute {PACKET_PATH}` to implement.
```

Or if max double-checks:

```
⚠️ **Packet checked:** {PACKET_PATH}

Check: max-double-checks ⚠️ ({N} total iterations, {R} restarts)
Confidence: max-double-checks — consider manual review

Next: Run `/ManageImpStep check {PACKET_PATH}` for manual review.
```

## Report

After execution, always include:

1. Status (complete or updated)
2. If updated: summary of additions
3. Confidence info (when `--loop` or `--double-check` used)
4. Suggested next command

## Auto-Suggest

After a successful check, auto-suggest the next command:

If confidence is `double-checked`:
```
/ManageImpStep execute {PACKET_PATH} -n
```

If confidence is `converged`:
```
/ManageImpStep execute {PACKET_PATH} -n
```

If confidence is `max-iterations` or `max-double-checks`:
```
/ManageImpStep check {PACKET_PATH} -n
```

If no flags used (single-pass):
```
/ManageImpStep execute {PACKET_PATH} -n
```

Replace `{PACKET_PATH}` with the actual packet path used in this run.

## Idempotency Note

This workflow is idempotent — running it multiple times will not duplicate information. Each run:

1. Re-reads source documents
2. Checks what's missing compared to current packet state
3. Only adds what's not already present
