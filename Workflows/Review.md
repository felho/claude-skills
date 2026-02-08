---
description: Review a PRD or structured spec for inconsistencies using parallel expert agents. Adapts to document size.
argument-hint: <prd-path> [<scope>]
allowed-tools: Read, Write, Glob, Grep, AskUserQuestion, Task
---

# Review — Parallel Consistency Check for PRD or Structured Spec

Launch parallel expert agents to review a PRD (monolithic or structured) for inconsistencies, gaps, and technical issues. Each agent focuses on a specific consistency dimension with fresh context. Adapts reading strategy based on document size to stay within context limits. Findings are merged, deduplicated, and reported by severity.

## Variables

PRD_PATH: $1 (path to the main PRD or design doc)
SCOPE: $2 (optional: "full" | "quick" — default: "full")
AGENT_MODEL: opus (for review agents — thorough, reliable tool use, low hallucination rate)
MIN_AGENT_READS: 5 (minimum Read tool calls an agent must make for findings to be valid)

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide a path to the PRD or design doc."
- **Path resolution (MUST do first):** `PRD_PATH` may be relative. Resolve it to an absolute path before ANY tool call:
  1. If `PRD_PATH` is already absolute (starts with `/`) → use as-is.
  2. If relative → resolve against the **current working directory** (the directory Claude Code was launched from). Use `Glob` with the relative pattern to find the actual file. If Glob finds exactly one match, use that absolute path. If zero matches, try common prefixes (`~/`, `./`) before reporting "not found".
  3. **Update `PRD_PATH`** to the resolved absolute path for all subsequent steps.
- This workflow is **read-only** — it does NOT modify the PRD or source files. It produces a findings report (optionally written to a file).
- Launch **all 7 agents in parallel** using the Task tool with `subagent_type: "general-purpose"`.
- If `SCOPE` is "quick", launch only agents 1-3 (the most impactful checks).
- Each agent gets the PRD path and any referenced code file paths. The agent reads the files itself.
- Focus on **actionable findings** — things that would cause bugs or confusion during implementation.
- **Context Budget Rule:** The orchestrating agent (you) must **NEVER** read the full PRD with the Read tool. Only scan structure using Grep. Each review agent reads file content in its own independent context window. Reading a large PRD here leaves no room for collecting 7 agents' results.

## Error Messages (Use Exactly)

- Missing PRD path: `"Usage: provide a path to the PRD or design doc."`
- File not found: `"PRD not found: {PRD_PATH}"`

## Workflow

### 1. Scan Structure (Do NOT Read Full File)

> **Do NOT use Read on the full PRD.** Reading a large file here consumes the context needed for collecting agent results. Agents read files in their own context windows.

1. **Verify file exists**: `Read` with `limit: 30` — read ONLY the first 30 lines for title and overview context.
   - If not found → STOP with "PRD not found: {PRD_PATH}"
2. **Count total lines**: `Grep` with pattern `"."` and `output_mode: "count"` on the PRD file.
3. **Build section map**: `Grep` with pattern `^#{1,3} ` and `output_mode: "content"` with `-n: true` to get all headings with line numbers.
4. **Detect structure type**:
   - **Monolithic:** Single markdown file, no external code references
   - **Structured:** References external files (look for paths like `packages/`, `src/`, backtick-quoted `.ts`/`.md` file paths)
5. **If structured**: Find referenced file paths using Grep/Glob. Resolve relative paths (e.g., `packages/domain/types.ts`) against the PRD file's **parent directory**, not the working directory. Count lines in all code files using **parallel Grep calls** (one `Grep` with `output_mode: "count"` per file, all in one message) — do NOT count files one at a time with separate Bash commands.
6. **Build FILE_LIST**: `[PRD_PATH, ...referenced_code_files]`
7. **Build CODE_FILE_MAP** (structured PRDs only): For each referenced code file, record:
   - File path
   - Line count
   - Brief content hint (infer from filename: `types.ts` → "type/enum definitions", `state-machine.ts` → "state transitions", `errors.ts` → "error codes", etc.)
8. **Calculate TOTAL_LINES** across all files. Also calculate **CODE_LINES** (sum of code file lines only) and **PRD_LINES** (PRD file lines only).
9. **Select review mode**:
   - **Standard** (TOTAL_LINES < 1000): Agents read full files directly
   - **Targeted** (TOTAL_LINES ≥ 1000): Agents receive section map + code file map and read selectively

Report to user:
```
Starting review of: <PRD_PATH>
Type: <Monolithic | Structured>
Lines: <TOTAL_LINES> → <standard | targeted> mode
<If structured:> Referenced files: <count> files found
Scope: <full | quick>
Launching <7 | 3> parallel review agents...
```

### 2. Launch Parallel Review Agents

#### Agent roster

| # | Agent file | Quick scope |
|---|------------|-------------|
| 1 | `agents/TypeConsistencyChecker.md` | yes |
| 2 | `agents/StateMachineChecker.md` | yes |
| 3 | `agents/ApiContractChecker.md` | yes |
| 4 | `agents/CrossReferenceChecker.md` | no |
| 5 | `agents/CompletenessChecker.md` | no |
| 6 | `agents/ArchitecturalChecker.md` | no |
| 7 | `agents/BuildVsReuseAdvisor.md` | no |

If `SCOPE` is "quick" → use only agents 1-3.

#### Prompt composition

First, **read ALL agent files in parallel** (all Read calls in one message). Then compose prompts from the results:

For each agent file:
<prompt-composition-loop>
1. Extract `keywords` from YAML frontmatter (already read above).
2. **Replace** `{FILE_LIST}` in the agent content with the actual file list from Step 1.
3. **If targeted mode** (TOTAL_LINES ≥ 1000) → **prepend** the appropriate Targeted Reading Prefix (see below) before the agent content. Fill in ALL template variables:

   | Variable | Source |
   |----------|--------|
   | `{TOTAL_LINES}` | Calculated in Step 1.8 |
   | `{FILE_COUNT}` | Number of files in FILE_LIST |
   | `{SECTION_MAP_FROM_STEP_1}` | Grep output from Step 1.3 (headings with line numbers) |
   | `{CODE_FILE_MAP}` | Built in Step 1.7 (structured PRDs only) |
   | `{AGENT_KEYWORDS}` | Agent's frontmatter `keywords` field (use `"*"` as "all sections") |
   | `{READING_BUDGET}` | Calculated per formula below |
   | `{MIN_READS}` | `MIN_AGENT_READS` variable (= 5) |
</prompt-composition-loop>

Store each composed prompt. Do NOT launch yet.

#### Targeted Reading Prefix templates

**`{READING_BUDGET}` calculation**:
- **Monolithic**: `min(2000, TOTAL_LINES * 0.5)`
- **Structured**: `min(2000, PRD_LINES * 0.5)` — budget applies ONLY to PRD lines, code files are read in full and don't count toward budget.

**`{CODE_FILE_MAP}` format** (built from Step 1):
```
- {file_path} ({line_count} lines) — {content_hint}
```

**For monolithic PRDs** (no companion code files):

```
CONTEXT: This is a large document ({TOTAL_LINES} lines).
Use targeted reading to stay within context limits.

Section map of {PRD_PATH}:
{SECTION_MAP_FROM_STEP_1}

YOUR PRIORITY KEYWORDS: {AGENT_KEYWORDS}

═══════════════════════════════════════════════════════════════════
MANDATORY READING PHASE — Complete BEFORE producing any findings.
The section map above is a NAVIGATION AID, not content.
You MUST use the Read tool to see actual file contents.
═══════════════════════════════════════════════════════════════════

READING STRATEGY:
1. Identify sections from the map above that match your priority keywords.
2. Read those sections using the Read tool with offset and limit parameters.
   Example: Read(file_path: "...", offset: 89, limit: 90) for lines 89-178.
3. If a finding references another section, read that section too.
4. Stay under {READING_BUDGET} lines total read.
5. ONLY AFTER reading, produce at most 10 findings, highest severity first.

ZERO-READ GUARD: If you have not made at least {MIN_READS} Read tool calls,
you do NOT have enough information to produce findings. Read more first.

ANTI-HALLUCINATION RULES:
- Do NOT cite line numbers you haven't seen in Read tool output.
- Do NOT claim a type/field/enum exists unless you saw it in Read output.
- Do NOT claim something is "missing" unless you read the file where it
  SHOULD be and confirmed it's absent.
- Fewer accurate findings are far better than many speculative ones.

SEVERITY CALIBRATION:
- Critical: Would cause a runtime bug, data loss, or crash during implementation.
- Important: Would cause confusion or incorrect implementation, but not a crash.
- Minor: Cosmetic, naming, or documentation-only issue.
```

**For structured PRDs** (with companion code files):

```
CONTEXT: This is a large structured document ({TOTAL_LINES} lines across {FILE_COUNT} file(s)).
The prose design doc references companion code files that contain the AUTHORITATIVE formal definitions (types, state machine, error codes, etc.).
Use targeted reading to stay within context limits.

PRD section map of {PRD_PATH}:
{SECTION_MAP_FROM_STEP_1}

Companion code files (SOURCE OF TRUTH for formal definitions):
{CODE_FILE_MAP}

YOUR PRIORITY KEYWORDS: {AGENT_KEYWORDS}

═══════════════════════════════════════════════════════════════════
MANDATORY READING PHASE — Complete BEFORE producing any findings.
The section map and code file map above are NAVIGATION AIDS, not
content. You MUST use the Read tool to see actual file contents.
Do NOT produce findings based on file names or section titles alone.
═══════════════════════════════════════════════════════════════════

READING STRATEGY — CODE FILES FIRST:
1. Read ALL companion code files listed above IN FULL using the Read tool.
   They are small (typically 100-400 lines each) and contain the
   authoritative definitions. This is your FIRST and MOST IMPORTANT step.
   Make one Read call per file. Do NOT skip any code file.
2. Then identify PRD sections from the map above that match your keywords.
3. Read those PRD sections using the Read tool with offset and limit.
   Example: Read(file_path: "...", offset: 89, limit: 90) for lines 89-178.
4. If a finding references another section, read that section too.
5. Stay under {READING_BUDGET} lines total read from the PRD file.
   Code files do NOT count toward this budget.
6. ONLY AFTER reading, produce at most 10 findings, highest severity first.

ZERO-READ GUARD: If you have not made at least {MIN_READS} Read tool calls,
you do NOT have enough information to produce findings. Read more first.

ANTI-HALLUCINATION RULES:
- Do NOT cite line numbers you haven't seen in Read tool output.
- Do NOT claim a type/field/enum value exists unless you saw it in Read output.
- Do NOT claim something is "missing" unless you read the file where it
  SHOULD be and confirmed it's absent.
- If unsure whether something exists, READ THE FILE before claiming.
- Fewer accurate findings are far better than many speculative ones.

SEVERITY CALIBRATION:
- Critical: Would cause a runtime bug, data loss, or crash during implementation.
- Important: Would cause confusion or incorrect implementation, but not a crash.
- Minor: Cosmetic, naming, or documentation-only issue.

VERIFY BEFORE REPORTING:
Before reporting ANY finding about missing, incomplete, or inconsistent definitions:
- CHECK the companion code files you read in step 1. Code files are SOURCE OF TRUTH.
- If a code file contains the correct, complete definition and the PRD prose
  references it, this is NOT a finding.
- Only report if: (a) genuinely missing from BOTH prose AND code, or
  (b) prose CONTRADICTS code, or (c) the issue is about the prose-code
  relationship itself (e.g., prose references a type that doesn't exist in code).
```

#### Parallel launch (background tasks)

> **⚠️ CRITICAL — USE BACKGROUND TASKS FOR PARALLELISM:**
> Launch ALL agents with `run_in_background: true`. Each Task call returns immediately with an `output_file` path — the agent runs concurrently in the background. This ensures all 7 agents execute in parallel regardless of whether the Task calls are emitted in one message or sequentially.

Launch all agents (7 or 3) as background Task calls:
- `subagent_type: "general-purpose"`
- `model: {AGENT_MODEL}`
- `run_in_background: true`
- `description:` agent title from the file (e.g., "Type Consistency Checker")

Save the returned `output_file` path for each agent. You will need these in Step 3.

### 3. Collect and Merge Results

After launching all agents, collect results by reading each agent's `output_file` using the Read tool. If an agent is still running, the output file will contain partial results — wait briefly and re-read.

1. **Read** each agent's output file. If the file ends without findings (agent still running) → wait 10 seconds (`Bash: sleep 10`) and re-read.
2. **Zero-read filter:** Check each agent's `tool_uses` count from the task notification. If an agent made fewer than `{MIN_AGENT_READS}` Read calls, **discard all its findings** and note it as "Discarded (insufficient reading)" in the Agent Summary. This catches agents that hallucinated findings from the prompt alone.
3. **Collect** all findings from agents that passed the zero-read filter.
4. **Deduplicate** — if two agents found the same issue (same location, same problem), merge into one finding, noting which agents found it.
5. **Number** findings sequentially (#1, #2, #3...).
6. **Group** by severity: Critical → Important → Minor.

### 4. Generate Report

Write the merged report to the user (and optionally to a file if the PRD is large).

## Report

```
PRD Review: <PRD name>
Type: <Monolithic | Structured>
Lines: <TOTAL_LINES> (<standard | targeted> mode)
Scope: <full | quick>
Agents: <7 | 3> launched, <N> returned findings

---

## Summary

<1-3 sentence high-level assessment>

The most critical issues are: <1-2 sentence summary of critical findings, if any>

| Severity | Count |
|----------|-------|
| Critical | <N> |
| Important | <N> |
| Minor | <N> |
| **Total** | **<N>** |

---

## Findings

### #1 — <Finding title> [Critical]

<Description of the inconsistency or issue>

**Where:** <section/file and line reference>
**Found by:** <agent name(s)>
**Impact:** <what goes wrong if not fixed>
**Recommendation:** <specific fix>

---

### #2 — <Finding title> [Important]
...

---

## Recommended Fix Order

1. <Most impactful fix first — finding #N>
2. <Second — finding #N>
...

## Agent Summary

| Agent | Reads | Findings | Status |
|-------|-------|----------|--------|
| Type Consistency Checker | <N> | <N> | Completed |
| State Machine Checker | <N> | <N> | Completed |
| API Contract Checker | <N> | <N> | Completed |
| Cross-Reference Checker | <N> | <N> | Completed |
| Completeness Checker | <N> | <N> | Completed |
| Architectural Checker | <N> | <N> | Completed |
| Build vs Reuse Advisor | <N> | <N> | Completed |
```

After the report, suggest: "To fix these findings, edit the PRD directly. After fixing, run the Review workflow again to verify."
