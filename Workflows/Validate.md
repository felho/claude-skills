---
description: Validate a structured PRD's code artifacts for cross-reference integrity, structural completeness, and type consistency using parallel checker agents.
argument-hint: <prd-path> [--scope=full|quick]
allowed-tools: Read, Glob, Grep, AskUserQuestion, Task
---

# Validate — Parallel Deterministic Check for Structured PRD Code Artifacts

Launch parallel validator agents to mechanically check a structured PRD's code files against explicit rules. Unlike the Review workflow (which uses expert judgment to find design issues), Validate runs a deterministic checklist — the same input always produces the same findings. Designed to run after Structure and before Review.

## Variables

PRD_PATH: $1 (path to the main PRD or design doc)
SCOPE: extracted from `--scope=` flag (optional: "full" | "quick" — default: "full")
AGENT_MODEL: opus (for validator agents — accuracy matters, mechanical checking needs reliable tool use)
MIN_AGENT_READS: 3 (minimum Read tool calls a validator must make for results to be valid)

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide a path to the PRD or design doc."
- **Path resolution (MUST do first):** `PRD_PATH` may be relative. Resolve it to an absolute path before ANY tool call:
  1. If `PRD_PATH` is already absolute (starts with `/`) → use as-is.
  2. If relative → resolve against the **current working directory**. Use `Glob` to find the actual file. If Glob finds exactly one match, use that absolute path. If zero matches, try common prefixes (`~/`, `./`) before reporting "not found".
  3. **Update `PRD_PATH`** to the resolved absolute path for all subsequent steps.
- This workflow is **read-only** — it does NOT modify the PRD or source files. It produces a findings report.
- The output format is **compatible with ProcessFindings** — findings use the same numbered format with severity, location, and recommendation.
- **Context Budget Rule:** The orchestrating agent (you) must **NEVER** read code files in full. Only use Grep/Glob for discovery. Each validator agent reads file content in its own independent context window.

## Error Messages (Use Exactly)

- Missing PRD path: `"Usage: provide a path to the PRD or design doc."`
- File not found: `"PRD not found: {PRD_PATH}"`

## Workflow

### 1. Discover (Sequential)

> **Do NOT use Read on code files.** Validator agents read files in their own context windows.

#### 1.1 Verify PRD Exists

`Read` with `limit: 20` — read ONLY the first 20 lines of the PRD for title and overview.
- If not found → STOP with "PRD not found: {PRD_PATH}"

#### 1.2 Find Code Files

Detect the project root (the PRD file's parent directory or its parent). Then:

1. **Glob** for all `.ts` files in the project directory tree.
2. **Glob** for all `.md` files (potential prose design docs).
3. Build **FILE_LIST**: the PRD path + all discovered `.ts` files + any other `.md` design docs.

#### 1.3 Classify Files by Pattern

For each `.ts` file, classify by filename and path:

| Pattern | File indicators | Validator |
|---------|----------------|-----------|
| State machine | `state-machine.ts`, `transitions.ts`, `fsm.ts` | Structural, CrossRef |
| Domain types | `types.ts`, `model.ts`, `domain.ts` | All three |
| Events | `events.ts`, `event-types.ts` | CrossRef |
| Commands | `commands.ts`, `command-types.ts` | CrossRef |
| Errors | `errors.ts`, `error-codes.ts` | Structural, CrossRef |
| Config | `config.ts`, `schema.ts` (in config dir) | CrossRef |
| DB Schema | `schema.ts` (in db dir), `*.schema.ts` | TypeConsistency |
| DTOs | `dto.ts`, `payloads.ts` | CrossRef, TypeConsistency |
| Rules/Logic | `rules.ts`, `invariants.ts`, `guards.ts` | CrossRef |
| Agent contracts | Files in `agents/` dir | CrossRef |

If a file doesn't match any known pattern, flag it in the discovery report as "uncovered."

#### 1.4 Determine Applicable Validators

Based on which patterns are present:

- **CrossReferenceIntegrity**: Always runs (at minimum checks import resolution and prose references).
- **StructuralCompleteness**: Runs if state machine file is detected.
- **TypeConsistency**: Runs if domain types file is detected.

If SCOPE is "quick" → run only CrossReferenceIntegrity.

#### 1.5 Report Discovery

```
Validation discovery: <PRD name>
Files: <N> code files + <M> doc files found
Patterns detected:
  - State machine: <filename or "not found">
  - Domain types: <filename or "not found">
  - Events: <filename or "not found">
  - Commands: <filename or "not found">
  - Errors: <filename or "not found">
  - Config: <filename or "not found">
  - DB Schema: <filename or "not found">

Uncovered files: <list of files not matching any pattern, or "none">
<If uncovered files exist:>
Note: These files are not covered by current validation rules.
Consider whether they contain cross-referenceable definitions.

Validators to run: <list based on patterns + scope>
Launching <N> parallel validators...
```

### 2. Launch Parallel Validators (Background Tasks)

#### Validator roster

| # | Validator file | Quick scope | Runs if |
|---|---------------|-------------|---------|
| 1 | `validators/CrossReferenceIntegrity.md` | yes | Always |
| 2 | `validators/StructuralCompleteness.md` | no | State machine detected |
| 3 | `validators/TypeConsistency.md` | no | Domain types detected |

#### Prompt composition

First, **read ALL applicable validator files in parallel** (all Read calls in one message). Then compose prompts:

For each validator file:
<prompt-composition-loop>
1. Read the validator file content.
2. **Replace** `{FILE_LIST}` with the actual file list from Step 1, formatted as:
   ```
   - <absolute_path> (<line_count> lines) — <pattern classification>
   ```
3. **Prepend** context about the PRD structure:
   ```
   You are validating: <PRD name>
   Project root: <path>

   Files to analyze:
   <FILE_LIST with classifications>

   MANDATORY: Read ALL code files listed above IN FULL before checking rules.
   Code files are small (typically 100-500 lines each).
   Make one Read call per file. Do NOT skip any file.

   ZERO-READ GUARD: You must make at least {MIN_AGENT_READS} Read calls
   before producing any results. If you haven't read enough files, read more.

   ANTI-HALLUCINATION RULES:
   - Do NOT cite line numbers you haven't seen in Read tool output.
   - Do NOT claim something exists/doesn't exist unless you read the file.
   - Report SKIP for rules where the pattern is absent, not PASS.
   - Fewer accurate findings are far better than many speculative ones.
   ```
</prompt-composition-loop>

#### Parallel launch (background tasks)

> **CRITICAL — USE BACKGROUND TASKS FOR PARALLELISM:**
> Launch ALL validators with `run_in_background: true`. Each Task call returns immediately with an `output_file` path — the validator runs concurrently in the background.

Launch all applicable validators as background Task calls:
- `subagent_type: "general-purpose"`
- `model: {AGENT_MODEL}`
- `run_in_background: true`
- `description:` validator title from the file (e.g., "Cross-Reference Integrity")

Save the returned `output_file` path for each validator.

### 3. Collect and Merge Results

After launching all validators, collect results by reading each validator's `output_file`.

1. **Read** each validator's output file. If the file ends without results (validator still running) → wait 10 seconds (`Bash: sleep 10`) and re-read.
2. **Zero-read filter:** If a validator made fewer than `{MIN_AGENT_READS}` Read calls, **discard its results** and note it as "Discarded (insufficient reading)" in the Validator Summary.
3. **Collect** all findings from validators that passed the filter.
4. **Flatten** findings from all validators into a single list.
5. **Deduplicate** — if two validators found the same issue (same file, same line, same problem), merge into one finding, noting which validators found it.
6. **Number** findings sequentially (#1, #2, #3...).
7. **Sort** by severity: Critical → Important → Minor.

### 4. Generate Report

Write the merged report to the user. If findings count > 10, also offer to write to a file.

## Report

```
PRD Validation: <PRD name>
Files: <N> code + <M> docs (<TOTAL_LINES> total lines)
Scope: <full | quick>
Validators: <N> launched, <N> returned results

---

## Rule Summary

| Validator | Rule | Status |
|-----------|------|--------|
| CrossRef | 1. Import Resolution | PASS/FAIL(N)/SKIP |
| CrossRef | 2. Emit → Events | PASS/FAIL(N)/SKIP |
| CrossRef | 3. Guards → Functions | PASS/FAIL(N)/SKIP |
| CrossRef | 4. Config References | PASS/FAIL(N)/SKIP |
| CrossRef | 5. Command → ACK | PASS/FAIL(N)/SKIP |
| CrossRef | 6. Prose References | PASS/FAIL(N)/SKIP |
| Structural | 1. Transition Fields | PASS/FAIL(N)/SKIP |
| Structural | 2. Side Effects Coverage | PASS/FAIL(N)/SKIP |
| Structural | 3. Error → Code | PASS/FAIL(N)/SKIP |
| Structural | 4. Error Annotations | PASS/FAIL(N)/SKIP |
| Structural | 5. Terminal States | PASS/FAIL(N)/SKIP |
| TypeConsist | 1. No Inline Literals | PASS/FAIL(N)/SKIP |
| TypeConsist | 2. Branded Types | PASS/FAIL(N)/SKIP |
| TypeConsist | 3. Schema Alignment | PASS/FAIL(N)/SKIP |

Rules: <passed> passed, <failed> failed, <skipped> skipped

---

## Summary

<1-3 sentence assessment. Focus on which rule categories have issues.>

| Severity | Count |
|----------|-------|
| Critical | <N> |
| Important | <N> |
| Minor | <N> |
| **Total** | **<N>** |

---

## Findings

### #1 — <Finding title> [Critical]

**Rule:** <Validator name — Rule N: Rule name>
**Where:** <file path and line reference>
**Found by:** <validator name(s)>
**Evidence:** <quoted code>
**Impact:** <what goes wrong if not fixed>
**Recommendation:** <specific fix>

---

### #2 — ...

---

## Recommended Fix Order

1. <Most impactful fix first — finding #N>
2. <Second — finding #N>
...

## Validator Summary

| Validator | Reads | Findings | Status |
|-----------|-------|----------|--------|
| Cross-Reference Integrity | <N> | <N> | Completed |
| Structural Completeness | <N> | <N> | Completed / Skipped |
| Type Consistency | <N> | <N> | Completed / Skipped |

---

<If findings > 0:>
To fix these findings, run: /CraftPRD process-findings <PRD_PATH>

<If findings == 0:>
All validation rules passed. Consider running /CraftPRD review for expert-level design analysis.
```

## STOP

**Do NOT continue past this point.** Report the findings and wait for the user.
