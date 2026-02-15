---
name: ValidateStep
description: Verify implementation meets acceptance criteria from packet
argument-hint: <packet-path> [--agents N]
allowed-tools: Read, Write, Bash, Glob, Grep, Task
# Note: Write is ONLY for the findings file, never for implementation files
# Note: Task is ONLY for multi-agent mode (--agents > 1). In single-agent mode (default), NEVER use Task.
---

# Validate Step Implementation

## Purpose

Verify that the implementation meets ALL acceptance criteria defined in the step packet. Supports two validation modes:

- **Single-agent mode** (default, `--agents 1`): Runs all checks inline in the main agent. Fast and sufficient for most steps.
- **Multi-agent mode** (`--agents N` where N > 1): Launches N parallel validation agents to reduce the risk of missed findings due to LLM non-determinism. Results are merged using a union strategy: if ANY agent finds an issue, it counts as a finding.

## Variables

PACKET_PATH: $1
FINDINGS_PATH: PACKET_PATH with `.md` replaced by `.findings.md`
Example: `decide-add-mapping.md` → `decide-add-mapping.findings.md`

PARALLEL_AGENTS: Parse from `--agents N` flag in arguments. Default: 1.

- If 1 → single-agent mode (inline validation, no Task tool needed)
- If >1 → multi-agent mode (launch N parallel validation agents via Task tool)

KNOWN_SIDE_EFFECTS:

- `.claude/settings.local.json` — Permission changes during session
- `bun.lock` — Dependency lock file updates
- `package-lock.json` — NPM lock file updates
- `yarn.lock` — Yarn lock file updates
- `*.findings.md` — Validation findings (transient, managed by validate/fix cycle)
- Other step packets in the same `*-steps/` directory — Pre-generated packets for upcoming steps (created by prepare workflow). These belong to the plan, not to this step. Ignore them during git status audit.

TEMP_ARTIFACT_PATTERNS:

- `coverage/` — Test coverage reports (gitignore)
- `*.tmp`, `*.temp` — Temporary files (delete)
- `*.log` — Log files (delete or gitignore)
- `.nyc_output/` — NYC coverage output (gitignore)

## Instructions

### Required YAML Frontmatter Fields

- `step`: Full step ID (phase-id/step-id)
- `plan`: Path to implementation plan
- `design-doc`: Path to design document
- `created`: ISO-8601 timestamp

### Validation Philosophy

**Thorough verification, not re-implementation.** This workflow checks that what was built meets the specification — it does not fix issues or write code.

### Validation Strategy

**Single-agent mode** (`PARALLEL_AGENTS = 1`): The main agent (you) runs all checks directly — type check, tests, file existence, acceptance criteria, and git status audit. No subagents are launched.

**Multi-agent mode** (`PARALLEL_AGENTS > 1`): LLMs are non-deterministic — a single validation pass may miss issues that another pass catches. To mitigate this:

- Launch `PARALLEL_AGENTS` independent validation agents, each reviewing the same packet
- Each agent runs type checks, tests, file existence, and acceptance criteria verification independently
- Merge results using **union strategy**: if ANY agent reports a criterion as FAIL, it is FAIL in the final result
- The main agent (you) handles deterministic checks (input validation, git status audit) and merges agent results

### Check Categories

1. **Type Check** — TypeScript compiles without errors
2. **Test Suite** — All tests pass
3. **File Existence** — Required files exist
4. **Acceptance Criteria** — Each criterion from packet is verified
5. **Git Status Audit** — Only expected deliverables in git status (no unexpected files)

### Error Messages (Use Exactly)

- Packet not found: `"Packet not found: {path}"`
- Invalid frontmatter: `"Invalid packet: missing required frontmatter fields (step, plan, design-doc)"`
- Plan not found: `"Plan file not found: {path}"`
- Step not in-progress (no status): `"Step {step-id} is not in-progress (status may have been manually removed). Run /ManageImpStep prepare to resume."`
- Step already done: `"Step {step-id} is already done. Remove status: done from the HTML comment, then run /ManageImpStep prepare."`

## Workflow

### Phase A: Pre-Validation (Main Agent)

These steps are deterministic and run once in the main agent.

### 1. Validate Input

- If `PACKET_PATH` is empty → STOP and report: `"Usage: /ManageImpStep validate <packet>"`
- Read packet file at `PACKET_PATH`
  - If file not found → STOP with "Packet not found" error
- Store the full packet content as `PACKET_CONTENT`

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

### 4. Determine App Directory

Identify the app directory for running type checks and tests:

- Look at the packet's implementation notes or file paths to determine the app directory
- This is typically the directory containing `package.json` and `tsconfig.json`
- Store as `APP_DIR`

### Phase B: Validation Checks

> **⚠️ GATE CHECK:** What is the value of PARALLEL_AGENTS?
> - If 1 (default): You MUST run all checks inline using Read, Grep, Glob, and Bash. **DO NOT use the Task tool.** Proceed to "If PARALLEL_AGENTS = 1" section.
> - If >1: Proceed to "If PARALLEL_AGENTS > 1" section. Task tool is required.

The execution path depends on `PARALLEL_AGENTS`.

<validation-checks>
The following four checks apply in BOTH modes. In single-agent mode, the main agent runs them directly. In multi-agent mode, each subagent runs them independently.

**Check 1: Type Check**

Run in the app directory (`APP_DIR`):

```bash
cd {APP_DIR} && bunx tsc --noEmit
```

Record: PASS or FAIL with error details.

**Check 2: Test Suite**

Run in the app directory (`APP_DIR`):

```bash
cd {APP_DIR} && bun run test
```

Record: PASS or FAIL. Note total passed/total tests and any failing test names.

**Check 3: File Existence**

For each file mentioned in the Step Definition section of the packet, verify the file exists using the Glob or Read tool. Record each as: exists or missing.

**Check 4: Acceptance Criteria Verification**

This is the most important check. For EACH numbered criterion in the "Acceptance Criteria" section of the packet:

1. Read the relevant source files to verify the criterion is met
2. Cross-reference with test results where applicable
3. Check that error messages match exact format from the spec
4. Verify edge cases mentioned in the packet
5. Check ALL sections of the packet — not just "Acceptance Criteria" but also "Step Definition", "Test Cases", "Review Step Completion", "Implementation Notes", and any other sections that describe expected behavior

**Be thorough:** Read the actual source code. Don't just trust that tests cover everything — verify the implementation logic directly.
</validation-checks>

---

#### If PARALLEL_AGENTS = 1 (Single-Agent Mode)

### 5. Run Validation Checks Inline

> **⚠️ CRITICAL:** In single-agent mode, you (the main agent) run ALL checks directly using Read, Grep, Glob, and Bash tools. **DO NOT launch Task agents.** Do NOT use the Task tool at all. Read the source files yourself and verify each criterion inline.

Run all four checks from `<validation-checks>` directly:

1. **Type Check** — Run `bunx tsc --noEmit` via Bash
2. **Test Suite** — Run `bun run test` via Bash
3. **File Existence** — Use Glob/Read to verify files exist
4. **Acceptance Criteria** — Use Read to open source files and verify each criterion yourself. Do NOT delegate this to a subagent.

Then proceed to step 7 (Git Status Audit).

---

#### If PARALLEL_AGENTS > 1 (Multi-Agent Mode)

### 5. Launch Parallel Validation Agents

Launch `PARALLEL_AGENTS` Task agents **in a single message** (all in parallel) using `subagent_type: "general-purpose"`. Each agent receives the same prompt constructed from the template below.

**IMPORTANT:** All agents MUST be launched in a single message to ensure true parallel execution. Do NOT launch them sequentially.

<validation-agent-prompt>
You are a validation agent. Your job is to independently verify that an implementation meets its acceptance criteria. Be thorough and skeptical — look for issues that might be easy to miss.

## Step Packet

```
{PACKET_CONTENT}
```

## Your Task

Verify the implementation against the acceptance criteria in the packet above. Work through each check category independently.

### Check 1: Type Check

Run in the app directory (`{APP_DIR}`):

```bash
cd {APP_DIR} && bunx tsc --noEmit
```

Record: PASS or FAIL with error details.

### Check 2: Test Suite

Run in the app directory (`{APP_DIR}`):

```bash
cd {APP_DIR} && bun run test
```

Record: PASS or FAIL. Note total passed/total tests and any failing test names.

### Check 3: File Existence

For each file mentioned in the Step Definition section of the packet, verify the file exists using the Glob or Read tool. Record each as: exists or missing.

### Check 4: Acceptance Criteria Verification

This is the most important check. For EACH numbered criterion in the "Acceptance Criteria" section of the packet:

1. Read the relevant source files to verify the criterion is met
2. Cross-reference with test results where applicable
3. Check that error messages match exact format from the spec
4. Verify edge cases mentioned in the packet
5. Check ALL sections of the packet — not just "Acceptance Criteria" but also "Step Definition", "Test Cases", "Review Step Completion", "Implementation Notes", and any other sections that describe expected behavior

**Be thorough:** Read the actual source code. Don't just trust that tests cover everything — verify the implementation logic directly.

### Output Format

Return your results in EXACTLY this format:

```
TYPE_CHECK: PASS|FAIL
TYPE_CHECK_DETAILS: {error details if FAIL, "No errors" if PASS}

TEST_SUITE: PASS|FAIL
TEST_SUITE_DETAILS: {passed}/{total} tests passing
TEST_FAILURES: {list of failing tests, or "None"}

FILE_EXISTENCE: PASS|FAIL
MISSING_FILES: {list of missing files, or "None"}

CRITERIA_RESULTS:
#1: PASS|FAIL|MANUAL — {criterion description}
  DETAILS: {what you verified and how}
#2: PASS|FAIL|MANUAL — {criterion description}
  DETAILS: {what you verified and how}
...continue for all criteria...

ADDITIONAL_FINDINGS:
{Any issues found that don't map to a specific criterion number, or "None"}
```

**IMPORTANT:** Number the criteria to match the acceptance criteria numbering in the packet. If the packet has criteria as checkboxes without explicit numbers, number them in order (1, 2, 3...).
</validation-agent-prompt>

### 6. Merge Agent Results

After all `PARALLEL_AGENTS` agents complete, merge their results:

**Merge rules:**

- **Union strategy:** If ANY agent reports a criterion as FAIL → it is FAIL in the merged result
- **Type check:** FAIL if any agent reports FAIL
- **Test suite:** FAIL if any agent reports FAIL (use the most detailed failure report)
- **File existence:** FAIL if any agent reports missing files
- **Acceptance criteria:** For each criterion number, FAIL if ANY agent marked it FAIL
- **Additional findings:** Collect all unique additional findings from all agents

**Deduplication:** When multiple agents report the same criterion as FAIL, keep the most detailed description. When agents describe the same issue with different wording, consolidate into one finding.

**Report agent agreement:** In the final output, note how many agents agreed on each finding. Example: "Found by 2/3 agents" or "Found by 1/3 agents" — this helps assess confidence.

---

### Phase C: Post-Validation (Main Agent)

### 7. Git Status Audit

**Purpose:** Ensure no unexpected files will be committed. Distinguish deliverables from interim artifacts.

Run `git status --short` and categorize each file:

<git-status-check-loop>
- If file is listed in packet's Step Definition or Acceptance Criteria → OK (expected deliverable)
- If file matches `KNOWN_SIDE_EFFECTS` patterns → OK (handle normally)
- If file is another step's packet in the same `*-steps/` directory → OK (pre-generated by prepare workflow, not this step's concern)
- If file matches `TEMP_ARTIFACT_PATTERNS` → clean up (delete or gitignore)
- If file is a verification-only test file → delete
- Otherwise → STOP with "Unexpected files detected" prompt (see below)
</git-status-check-loop>

**Detecting verification-only test files:**

A test file is a "verification artifact" (not a deliverable) if ALL of these are true:

- Located in `tests/` directory
- Contains NO imports from the project's `src/` directory
- Only tests trivial assertions (infrastructure checks like "1+1=2", "directory exists")
- Is NOT listed as a deliverable in the packet's Acceptance Criteria

→ If detected: delete the file before proceeding

**If unknown files found → STOP:**

```
⚠️ Unexpected files detected:

The following files appeared but aren't listed as expected deliverables:
  - {file1}
  - {file2}

Options:
1. These are deliverables I forgot to mention (will be committed)
2. These are temporary and should be deleted
3. These should be added to .gitignore
4. Let me explain...

How should I handle these files?
```

**IMPORTANT:** Do NOT proceed to "Validation PASSED" if there are unknown files. The user must explicitly confirm how to handle them.

### 8. Compile Results

Aggregate all check results (from step 5 in single-agent mode, or merged results from step 6 in multi-agent mode) and git status audit from step 7:

- Type check: PASS/FAIL
- Tests: X/Y passing
- Files: all exist / N missing
- Acceptance criteria: X/Y verified, Z manual
- Git status audit: clean / has issues

Determine overall status:

- **PASS** — All automated checks pass AND git status is clean
- **FAIL** — Any automated check fails
- **PARTIAL** — Automated pass, but has manual verification items
- **BLOCKED** — Unknown files in git status need user input

### 9. Write or Delete Findings File

Based on the overall status determined in step 8:

- If **PASS** → delete `FINDINGS_PATH` if it exists (silently, no error if absent)
- If **FAIL**, **PARTIAL**, or **BLOCKED** → write structured findings to `FINDINGS_PATH`

**Findings file format:**

```markdown
---
step: { STEP_ID }
packet: { PACKET_PATH }
validated-at: { ISO-8601 timestamp }
result: FAIL|PARTIAL|BLOCKED
---

## Failed

### {criterion number}. {criterion description}

**Expected:** {what should be true based on the packet}
**Found:** {what was actually found during validation}
**Packet section:** {which section of the packet describes the expected behavior}

## Passed

1. {criterion description} ✓
2. {criterion description} ✓
   ...
```

**Rules:**

- Each failed criterion gets its own H3 subsection under `## Failed`
- Passed criteria are a numbered list under `## Passed`
- The `## Failed` section comes first (most important for the Fix workflow)
- If result is PARTIAL, manual-verification items go under `## Manual` (between Failed and Passed)
- If result is BLOCKED, the unknown files go under `## Blocked` (between Failed and Passed)
- The `**Packet section**` field helps the Fix workflow find the relevant specification without reading the entire packet

**Git status note:** The findings file (`*.findings.md`) is a transient artifact. It should NOT be committed — the Git Status Audit (step 9) should treat `*.findings.md` files as known side effects (same as `KNOWN_SIDE_EFFECTS` patterns).

## Report

### If ALL checks PASS:

> **⚠️ BEFORE reporting PASS:** Delete `FINDINGS_PATH` if it exists (from a previous failed run). This is Step 9 — do NOT skip it. Report the cleanup in the output below.

```
✅ Validation PASSED: {STEP_ID}

{If PARALLEL_AGENTS > 1: "Agents: {PARALLEL_AGENTS}/{PARALLEL_AGENTS} agree — no issues found"}
Type check: ✓ No errors
Tests: {passed}/{total} passing
Files: All required files exist
Acceptance criteria: {verified}/{total} verified
Git status: ✓ Clean (only expected deliverables)
{If findings file existed and was deleted: "Findings cleanup: ✓ Deleted stale findings file from previous run"}

Next: Run `/ManageImpStep done {PACKET_PATH}` to mark step complete.
```

### If ANY check FAILS or PARTIAL:

```
❌ Validation FAILED: {STEP_ID}

{If PARALLEL_AGENTS > 1: "Agents: {N}/{PARALLEL_AGENTS} found issues"}
Type check: {✓ or ✗ with error count}
Tests: {passed}/{total} passing
  - {failing test 1}
  - {failing test 2}
Files: {status}
  - Missing: {file1}, {file2}
Acceptance criteria: {verified}/{total}
  - ✗ {failed criterion} {If PARALLEL_AGENTS > 1: "(found by {M}/{PARALLEL_AGENTS} agents)"}
Git status: {✓ or ⚠️ if issues}

Findings written to: {FINDINGS_PATH}

Fix the issues and re-run `/ManageImpStep validate {PACKET_PATH}`,
or run `/ManageImpStep fix {PACKET_PATH}` for targeted fixes.
```

### If PARTIAL (has manual checks):

```
⚠️ Validation PARTIAL: {STEP_ID}

Automated checks: ✓ All passed
- Type check: ✓
- Tests: {passed}/{total}
- Files: ✓
- Git status: ✓ Clean

Manual verification needed:
- [ ] {criterion requiring manual check}
- [ ] {another manual criterion}

Findings written to: {FINDINGS_PATH}

If manual checks pass, run `/ManageImpStep done {PACKET_PATH}`.
If automated fixes are needed, run `/ManageImpStep fix {PACKET_PATH}`.
```

## Auto-Suggest

Auto-suggest the next command based on the validation result:

- **PASS** → `/ManageImpStep done {PACKET_PATH} -n`
- **FAIL** or **PARTIAL** → `/ManageImpStep fix {PACKET_PATH} -n`
- **BLOCKED** → do not auto-suggest (requires user decision)

Replace `{PACKET_PATH}` with the actual packet path used in this run.

### If BLOCKED (unknown files in git status):

```
⚠️ Validation BLOCKED: {STEP_ID}

Automated checks: ✓ All passed
- Type check: ✓
- Tests: {passed}/{total}
- Files: ✓

Git status audit: ⚠️ Unknown files detected

The following files appeared but aren't listed as expected deliverables:
  - {file1}
  - {file2}

Options:
1. These are deliverables I forgot to mention (will be committed)
2. These are temporary and should be deleted
3. These should be added to .gitignore
4. Let me explain...

How should I handle these files?
```
