---
description: Verify packet completeness against source documents, improve if needed
argument-hint: <packet> [--loop] [--double-check] [--orchestrate] [--dc-pass] [--max-cycles N] [--debug]
allowed-tools: Read, Edit, Glob, Bash, Task, Skill
# Note: Edit is ONLY for updating the packet .md file, never for implementation files
# Note: Bash is ONLY for running validator scripts explicitly
# Note: Task is for spawning background single-pass iterations (orchestrator mode)
# Note: Skill is for legacy --loop/--double-check flag mapping
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

> **SCOPE: This workflow ONLY modifies the packet `.md` file.**
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
MAX_CYCLES: from `--max-cycles N` (default: 3, only meaningful with `--orchestrate`)
DEBUG: from `--debug` flag (default: false)
DEBUG_LOG: `~/.claude/hooks/validators/ManageImpStep/check-debug.log`

**Mode flags** (mutually exclusive — pick the first match):

| Flag | Mode | Description |
|------|------|-------------|
| `--orchestrate` | Orchestrator | Spawns single-pass iterations as background Task agents, monitors convergence, handles Phase A->B flow |
| `--dc-pass` | DC-pass | One Phase B evaluation (7 dimensions), fixes if needed, reports scores |
| `--loop` | Legacy | Maps to `--orchestrate` (backwards compat) |
| `--double-check` | Legacy | Implies `--orchestrate` (backwards compat) |
| (none) | Single-pass | One iteration: read, analyze, fix, report |

**Modifier flags** (combinable with any mode):

| Flag | Description |
|------|-------------|
| `--debug` | Write structured entries to `DEBUG_LOG` at key checkpoints |

**Flag parsing:** Scan all arguments for flags. `--loop` and `--double-check` both map to orchestrator mode internally. `--debug` is independent and can be combined with any mode. Remove flags from positional arguments before processing.

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

## Shared Steps (All Modes)

### 1. Validate Input

- If `PACKET_PATH` is empty -> STOP and report: `"Usage: /ManageImpStep check <packet> [--orchestrate] [--dc-pass] [--max-cycles N]"`
- Read packet file at `PACKET_PATH`
  - If file not found -> STOP with "Packet not found" error

### 2. Parse Packet Frontmatter

Extract from YAML frontmatter:

- `step` -> STEP_ID
- `plan` -> PLAN_PATH
- `design-doc` -> DESIGN_DOC_PATH

If any required field is missing -> STOP with "Invalid packet" error

### 3. Verify Step Status in Plan

- Read plan file at `PLAN_PATH`
  - If file not found -> STOP with "Plan file not found" error
- Find the step's HTML comment: `<!-- id: {step-id} ... -->`
- Check status:
  - If no `status` attribute -> STOP with "Step not in-progress" error
  - If `status: done` -> STOP with "Step already done" error
  - If `status: in-progress` -> continue
  - If `status: prepared` -> continue (packet exists from prepare-ahead)

### 4. Route to Mode

After shared validation, route to the appropriate mode based on flags:

- `--orchestrate` (or `--loop` / `--double-check`) -> **Orchestrator Mode**
- `--dc-pass` -> **DC-Pass Mode**
- (no flags) -> **Single-Pass Mode**

---

## Debug Logging (when `--debug` is set)

When `DEBUG` is true, write structured log entries at key checkpoints using Bash:

```bash
echo "[$(date '+%Y-%m-%d %H:%M:%S')] [MODE] message" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log
```

Where `MODE` is one of: `single-pass`, `orchestrate`, `dc-pass`.

**All log writes are conditional on `--debug`.** If `--debug` is not set, skip all logging. The specific checkpoints are noted inline in each mode section below (marked with `🔍 DEBUG`).

**Propagation:** When spawning subagents (orchestrator → single-pass, orchestrator → dc-pass), include `--debug` in the Task prompt if `DEBUG` is true.

---

## Single-Pass Mode (default, no special flags)

One iteration: read source documents, analyze packet for gaps, fix, report.

### SP-1. Read Source Documents Thoroughly

> 🔍 **DEBUG:** `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [single-pass] Started for {PACKET_PATH}" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`

Read the design document at `DESIGN_DOC_PATH`:

- If file not found -> STOP with "Design document not found" error

Read the plan file at `PLAN_PATH`.

<reading-rules>
**Do NOT use the `limit` parameter** when reading source documents. The Read tool's default behavior reads the full file -- that is what you want.

**Anti-pattern (BLOCKED by hook):**
```
Read(file_path: "design.md", limit: 200)  <- WRONG: skips 90%+ of the document
```

**Correct pattern:**
```
Read(file_path: "design.md")              <- RIGHT: reads full file
```

**For very large files (>2000 lines):** use sequential offset reads to cover the entire file:
```
Read(file_path: "design.md")                        <- first 2000 lines
Read(file_path: "design.md", offset: 2000)           <- next chunk
Read(file_path: "design.md", offset: 4000)           <- and so on
```

**Self-check after reading:** Can you recall content from the LAST section of the document? If not, you didn't read far enough.
</reading-rules>

### SP-2. Analyze Packet for Missing Information

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

### SP-3. Capture Potential Issues (Interim List)

Create a numbered list of potential issues found during analysis. This is an interim working list -- items may turn out to be already covered.

**Format:**

```
**Potential Issues Found:**
1. [Description of what might be missing]
2. [Another potential gap]
3. [Edge case that may not be covered]
```

**Important:** This list captures your initial findings. Each item will be verified in the next step.

### SP-4. Verify Each Issue Against Packet

For each potential issue, search the packet for evidence. Provide explicit proof (line numbers, exact quotes) for your verification.

**Format:**

```
**Verification:**
1. [Issue description]
   -> Found at line X: "[exact quote from packet]"

2. [Issue description]
   -> Found at line Y: "[exact quote]"

3. [Issue description]
   -> Missing -- needs to be added to packet
```

**Rules:**

- Every potential issue MUST be verified with proof
- Use exact line numbers from the packet
- Quote the relevant text that proves coverage
- Mark clearly: Found (with proof) or Missing
- The final verdict must be consistent with verification results

### SP-5. Update Packet with Missing Information

> 🔍 **DEBUG:** After verification, log a summary: `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [single-pass] Findings: {N} potential, {M} missing, {F} fixed" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`
> Where N = total potential issues, M = confirmed missing, F = items fixed in packet.

For each item marked Missing in verification:

<update-rules>
- Add missing design doc content to "From Design Document" section
- Add missing plan content to "From Plan (Supporting Sections)" section
- Add missing test cases to "Test Cases" section
- Add missing acceptance criteria to "Acceptance Criteria" section
- Add missing gotchas/warnings to "Implementation Notes" section
- Do NOT duplicate information already in the packet
- Do NOT remove existing content
- Use the packet's existing section structure
</update-rules>

After each packet Edit, run the structure validator explicitly:

```bash
uv run $HOME/.claude/hooks/validators/ManageImpStep/packet-structure-validator.py
```

### SP-6. Report Result

**No changes needed:**

```
**Potential Issues Found:**
1. {brief description}
2. {brief description}

**Verification:**
1. {description} -> Found at line X: "{quote}"
2. {description} -> Found at line Y: "{quote}"

---

Packet is complete: {PACKET_PATH}

All potential issues verified as present in packet. Ready for implementation.

Next: Run `/ManageImpStep execute {PACKET_PATH}` to implement.
```

**Packet was updated:**

```
**Potential Issues Found:**
1. {brief description}
2. {brief description}
3. {brief description}

**Verification:**
1. {description} -> Found at line X: "{quote}"
2. {description} -> Missing
3. {description} -> Missing

---

Packet updated: {PACKET_PATH}

Added:
- {summary of what was added for issue #2}
- {summary of what was added for issue #3}

Next: Review changes, then run `/ManageImpStep execute {PACKET_PATH}` to implement.
```

---

## Orchestrator Mode (`--orchestrate`)

Spawns single-pass iterations as background Task agents, monitors convergence, then runs Phase B (dc-pass) for structured evaluation. Each iteration runs in its own context window to prevent context exhaustion.

> **This is a Level 4 (Delegate) prompt** -- it spawns and coordinates subagents.

### O-1. Initialize Tracking

> 🔍 **DEBUG:** `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [orchestrate] Started: max-iterations=10, max-cycles={MAX_CYCLES}" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`

```
total-iterations = 0
double-check-restarts = 0
MAX_ITERATIONS = 10
```

### O-2. Phase A Loop (Single-Pass Iterations)

Repeat until convergence or MAX_ITERATIONS:

<phase-a-loop>
1. **Spawn single-pass iteration:**
   > 🔍 **DEBUG:** `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [orchestrate] Phase A: spawning iteration {N}" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`

   ```
   Task(
     description: "Check packet iteration {N}",
     subagent_type: "general-purpose",
     run_in_background: true,
     prompt: "Use the Skill tool to invoke: /ManageImpStep check {PACKET_PATH} {--debug if DEBUG}"
   )
   ```

2. **Wait for completion:** Use `TaskOutput` to wait for the background agent to finish.

3. **Increment** `total-iterations`.

4. **Detect convergence:** Read the packet file. Compare content against previous iteration:
   - Compute a simple hash/checksum of the packet content, OR
   - Check the Task agent's output for "Packet is complete" (indicates zero findings)
   - If the single-pass agent reported no changes -> **converged**, exit Phase A
     > 🔍 **DEBUG:** `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [orchestrate] Phase A converged after {N} iterations" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`
   - If changes were made -> spawn another iteration
     > 🔍 **DEBUG:** `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [orchestrate] Iteration {N}: packet changed, continuing" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`

5. **Max iterations guard:** If `total-iterations` >= MAX_ITERATIONS -> exit Phase A as `max-iterations`
</phase-a-loop>

### O-3. Phase B (DC-Pass)

> If Phase A ended as `max-iterations`, skip Phase B. Set confidence to `max-iterations` and go to O-4.

After Phase A converges, spawn a DC-pass evaluation:

> 🔍 **DEBUG:** `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [orchestrate] Phase B: spawning dc-pass" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`

```
Task(
  description: "DC-pass evaluation",
  subagent_type: "general-purpose",
  run_in_background: true,
  prompt: "Use the Skill tool to invoke: /ManageImpStep check {PACKET_PATH} --dc-pass {--debug if DEBUG}"
)
```

Wait for completion. Read the packet frontmatter for dimension scores.

**Evaluate DC-pass result:**

- Read packet frontmatter for `dc-dimensions` field (set by DC-pass mode)
- If all 7 dimensions PASS -> set confidence to `double-checked`, STOP
- If any WEAK or MISSING:
  - Increment `double-check-restarts`
  - If `double-check-restarts` < MAX_CYCLES - 1:
    - Go back to O-2 (Phase A) to re-verify consistency after DC-pass fixes
  - If MAX_CYCLES exhausted:
    - Set confidence to `max-double-checks`, STOP

### O-4. Update Frontmatter with Confidence Metadata

> 🔍 **DEBUG:** `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [orchestrate] Done: {confidence} ({total-iterations} iterations, {double-check-restarts} restarts)" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`

Update the packet's YAML frontmatter:

- If converged + all 7 pass: `check-confidence: double-checked`, `check-iterations: {total}`, `double-check-restarts: {R}`
- If max double-checks: `check-confidence: max-double-checks`, `check-iterations: {total}`, `double-check-restarts: {MAX_CYCLES - 1}`
- If max iterations: `check-confidence: max-iterations`, `check-iterations: {total}`

Remove `check-confidence: unchecked` if present (replace with actual result).

### O-5. Report Result

**Double-checked (all pass):**

```
Packet checked: {PACKET_PATH}

Check: double-checked ({N} total iterations, {R} restarts)
Confidence: double-checked

| # | Dimension | Score |
|---|-----------|-------|
| 1 | Technical Sufficiency | PASS |
| 2 | Error Handling Coverage | PASS |
| ... | ... | ... |

Next: Run `/ManageImpStep execute {PACKET_PATH}` to implement.
```

**Max double-checks:**

```
Packet checked: {PACKET_PATH}

Check: max-double-checks ({N} total iterations, {R} restarts)
Confidence: max-double-checks -- consider manual review

Next: Run `/ManageImpStep check {PACKET_PATH}` for manual review.
```

**Max iterations (Phase A did not converge):**

```
Packet checked: {PACKET_PATH}

Check: did not converge after {N} iterations
Confidence: max-iterations -- manual review recommended

Next: Run `/ManageImpStep check {PACKET_PATH}` for manual review, then `/ManageImpStep execute {PACKET_PATH}`.
```

---

## DC-Pass Mode (`--dc-pass`)

One Phase B evaluation: structured 7-dimension assessment. Fixes issues if found, writes dimension scores to packet frontmatter. Designed to be invoked by the Orchestrator as a subagent.

### DC-1. Read Source Documents

Read the design document at `DESIGN_DOC_PATH` and the plan file at `PLAN_PATH`.

(Same reading-rules as Single-Pass SP-1.)

### DC-2. Read Double-Check Dimensions

Read the 7 semantic dimensions from `~/.claude/skills/ManageImpStep/references/DoubleCheckPrompt.md`.

### DC-3. Evaluate 7 Dimensions

> 🔍 **DEBUG:** After evaluation, log scores: `echo "[$(date '+%Y-%m-%d %H:%M:%S')] [dc-pass] Scores: tech={S1} err={S2} edge={S3} test={S4} dep={S5} ac={S6} impl={S7}" >> ~/.claude/hooks/validators/ManageImpStep/check-debug.log`
> Where S1-S7 are PASS/WEAK/MISSING for each dimension in order.

For each dimension, evaluate the packet:

1. **Technical Sufficiency** -- types, interfaces, configs, file paths all specified?
2. **Error Handling Coverage** -- all error cases with messages and recovery paths?
3. **Edge Cases & Validation** -- boundary conditions, validation rules, concurrency?
4. **Testability** -- test cases concrete enough to write actual test code?
5. **Dependency Clarity** -- inter-step deps and external deps explicit?
6. **Acceptance Criteria Clarity** -- each AC specific, measurable, binary (pass/fail)?
7. **Implementation Completeness** -- could implementer start with ONLY this packet?

For each dimension:
- Quote specific evidence from the packet
- Cross-reference against plan and design doc
- Score: **PASS** / **WEAK** / **MISSING**

### DC-4. Fix Weak/Missing Dimensions

For any dimension scored WEAK or MISSING:

- Add missing content from plan and design doc to the appropriate packet section
- Do NOT remove or duplicate existing content
- Use the packet's existing section structure

After each packet Edit, run the structure validator:

```bash
uv run $HOME/.claude/hooks/validators/ManageImpStep/packet-structure-validator.py
```

### DC-5. Update Frontmatter with Dimension Scores

Add/update `dc-dimensions` field in packet frontmatter with the evaluation results:

```yaml
dc-dimensions:
  technical-sufficiency: PASS
  error-handling: PASS
  edge-cases: WEAK
  testability: PASS
  dependency-clarity: PASS
  acceptance-criteria: PASS
  implementation-completeness: PASS
```

This field is read by the Orchestrator to determine whether to restart Phase A.

### DC-6. Report Result

```
DC-Pass evaluation: {PACKET_PATH}

| # | Dimension | Score | Evidence / Gap |
|---|-----------|-------|----------------|
| 1 | Technical Sufficiency | PASS/WEAK/MISSING | [quote or gap] |
| 2 | Error Handling Coverage | PASS/WEAK/MISSING | [quote or gap] |
| 3 | Edge Cases & Validation | PASS/WEAK/MISSING | [quote or gap] |
| 4 | Testability | PASS/WEAK/MISSING | [quote or gap] |
| 5 | Dependency Clarity | PASS/WEAK/MISSING | [quote or gap] |
| 6 | Acceptance Criteria Clarity | PASS/WEAK/MISSING | [quote or gap] |
| 7 | Implementation Completeness | PASS/WEAK/MISSING | [quote or gap] |

{If all PASS: "All dimensions pass."}
{If any WEAK/MISSING: "Fixes applied. Dimensions with gaps: {list}"}
```

---

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

This workflow is idempotent -- running it multiple times will not duplicate information. Each run:

1. Re-reads source documents
2. Checks what's missing compared to current packet state
3. Only adds what's not already present
