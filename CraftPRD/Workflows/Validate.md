---
description: Validate a structured PRD's code artifacts for cross-reference integrity, structural completeness, and type consistency using parallel checker agents.
argument-hint: <prd-path> [--scope=full|quick] [--replicas=name:count,...]
allowed-tools: Read, Glob, Grep, AskUserQuestion, Task
---

# Validate — Parallel Deterministic Check for Structured PRD Code Artifacts

Launch parallel validator agents to mechanically check a structured PRD's code files against explicit rules. Unlike the Review workflow (which uses expert judgment to find design issues), Validate runs a deterministic checklist — the same input always produces the same findings. Designed to run after Structure and before Review.

## Variables

PRD_PATH: $1 (path to the main PRD or design doc)
SCOPE: extracted from `--scope=` flag (optional: "full" | "quick" — default: "full")
AGENT_MODEL: opus (for validator agents — accuracy matters, mechanical checking needs reliable tool use)
MIN_AGENT_READS: 3 (minimum Read tool calls a validator must make for results to be valid)
REPLICAS: extracted from `--replicas=` flag (optional).
  Format: comma-separated `name:count` pairs.
  Short names: crossref, structural, type.
  Default: each applicable validator runs 1 instance (current behavior).
  Setting count to 0 skips that validator entirely.
  Example: `--replicas=structural:3,crossref:0,type:0`

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

#### 1.4 Determine Applicable Validators and Replica Counts

**Precedence rules** (first match wins):

1. **If `REPLICAS` is specified** → it takes full control. Each validator's instance count comes from the flag. Validators not mentioned in the flag default to 1 instance if applicable (per pattern rules below), or 0 if not applicable. A count of 0 skips that validator entirely. `--scope` is ignored when `--replicas` is present.

2. **If only `SCOPE` is "quick"** (no `--replicas`) → only CrossReferenceIntegrity runs (1 instance). Same as current behavior.

3. **If neither is specified** → default behavior: 1 instance per applicable validator (current behavior).

**Applicability** (used for defaults when a validator is not mentioned in `--replicas`):

- **CrossReferenceIntegrity** (`crossref`): Always applicable.
- **StructuralCompleteness** (`structural`): Applicable if state machine file is detected.
- **TypeConsistency** (`type`): Applicable if domain types file is detected.

Build a **VALIDATOR_PLAN**: a list of `(validator_name, instance_count)` tuples to use in subsequent steps.

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

Validators to run:
  - Cross-Reference Integrity: <N> instance(s) <or "skipped" if 0>
  - Structural Completeness: <N> instance(s) <or "skipped" if 0>
  - Type Consistency: <N> instance(s) <or "skipped" if 0>
Launching <TOTAL> parallel validators...
```

### 2. Launch Parallel Validators (Background Tasks)

#### Validator roster

| # | Validator file | Short name | Quick scope | Default applicability |
|---|---------------|------------|-------------|----------------------|
| 1 | `validators/CrossReferenceIntegrity.md` | crossref | yes | Always |
| 2 | `validators/StructuralCompleteness.md` | structural | no | State machine detected |
| 3 | `validators/TypeConsistency.md` | type | no | Domain types detected |

#### Prompt composition

First, **read ALL applicable validator files in parallel** (all Read calls in one message — read each validator file only once regardless of replica count). Then compose prompts:

For each validator file:
<prompt-composition-loop>
1. Read the validator file content (once per validator type, reuse for all replicas).
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

Each replica of the same validator type gets the **exact same prompt** — no differentiation. Replicas rely on LLM non-determinism to produce different findings.

#### Replica naming

- **1 instance** → use the validator name as-is (e.g., "Structural Completeness") — same as today.
- **N instances (N > 1)** → append a `#` suffix: "Structural Completeness #1", "Structural Completeness #2", etc.

#### Parallel launch (background tasks)

> **CRITICAL — USE BACKGROUND TASKS FOR PARALLELISM:**
> Launch ALL validator replicas with `run_in_background: true`. Each Task call returns immediately with an `output_file` path — the validator runs concurrently in the background.

<replica-launch-loop>
For each entry in VALIDATOR_PLAN where instance_count > 0:
  For replica_index from 1 to instance_count:
    Launch a background Task call:
    - `subagent_type: "general-purpose"`
    - `model: {AGENT_MODEL}`
    - `run_in_background: true`
    - `description:` replica display name (e.g., "Structural Completeness #2")
    - `prompt:` the composed prompt for this validator type (same for all replicas)

    Save the returned `output_file` path, keyed by replica display name.
</replica-launch-loop>

Validators with instance_count = 0 are skipped entirely (no Task launched).

### 3. Collect and Merge Results

After launching all validator replicas, collect results by reading each replica's `output_file`.

1. **Read** each replica's output file. If the file ends without results (replica still running) → wait 10 seconds (`Bash: sleep 10`) and re-read.
2. **Zero-read filter (per replica):** If a replica made fewer than `{MIN_AGENT_READS}` Read calls, **discard its results** and note it as "Discarded (insufficient reading)" in the Validator Summary. Each replica is filtered independently.
3. **Collect** all findings from replicas that passed the filter.
4. **Flatten** findings from all replicas into a single list.
5. **Deduplicate** — merge findings that match on (same file, same line, same problem):
   - **Across different validator types** (same as before): merge into one finding, noting which validators found it.
   - **Across replicas of the same validator type**: same rule — merge into one finding. The `Found by` field lists all replicas that independently found it (e.g., "Structural #1, Structural #3").
   - Different findings from different replicas → keep both as separate findings.
   - Track `total_findings` (before dedup) and `unique_findings` (after dedup) for the dedup note.
6. **Number** findings sequentially (#1, #2, #3...).
7. **Sort** by severity: Critical → Important → Minor.

### 4. Generate Report

Write the merged report to the user. If findings count > 10, also offer to write to a file.

## Report

```
PRD Validation: <PRD name>
Files: <N> code + <M> docs (<TOTAL_LINES> total lines)
Scope: <full | quick>
Replicas: <replica config summary, e.g. "structural:3, crossref:0, type:0" or "default (1 each)">
Validators: <N> replicas launched, <N> returned results

---

## Rule Summary

> Aggregate across replicas — FAIL(N) if **any** replica found issues for that rule.

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
| TypeConsist | 4. Cross-File Field Types | PASS/FAIL(N)/SKIP |

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

<If any replicas ran (total instance count > 1 for any validator):>
Unique findings: <unique_findings> (from <total_findings> total across <total_replicas> replicas, <duplicates_merged> duplicates merged)

---

## Recommended Fix Order

1. <Most impactful fix first — finding #N>
2. <Second — finding #N>
...

## Validator Summary

| Validator | Reads | Findings | Status |
|-----------|-------|----------|--------|
| Cross-Reference Integrity | <N> | <N> | Completed / Skipped (0 replicas) |
| Structural Completeness #1 | <N> | <N> | Completed |
| Structural Completeness #2 | <N> | <N> | Completed |
| ... | ... | ... | ... |
| Type Consistency | — | — | Skipped (0 replicas) |

> Show one row per replica. Use the replica display name (with `#N` suffix when N > 1).
> For skipped validators (0 replicas), show a single row with "—" for Reads/Findings.
> For discarded replicas (insufficient reads), show "Discarded (insufficient reading)" in Status.

---

<If findings > 0:>
To fix these findings, run: /CraftPRD process-findings <PRD_PATH>

<If findings == 0:>
All validation rules passed. Consider running /CraftPRD review for expert-level design analysis.
```

## STOP

**Do NOT continue past this point.** Report the findings and wait for the user.
