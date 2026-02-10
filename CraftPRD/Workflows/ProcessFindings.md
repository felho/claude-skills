---
description: Process review findings one-by-one with medium-aware thinking. Pipeline pattern — present next finding while committing previous fix.
argument-hint: <prd-path> [findings-file] [--min-severity=critical|important|minor]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, Task
---

# ProcessFindings — Pipeline Finding Resolution

Process review findings sequentially, applying the CraftPRD medium-aware philosophy at each step. Uses a pipeline pattern: while the user reads and decides on finding N, the previous fix (N-1) is committed in the background.

Designed to run after the Review workflow produces a findings report.

## Variables

PRD_PATH: $1 (path to the PRD or design doc being fixed)
FINDINGS_SOURCE: $2 (optional: path to findings file. If not provided, use the findings report from context — i.e., the Review workflow output earlier in this conversation.)
MIN_SEVERITY: extracted from `--min-severity=` flag in arguments (default: "minor" — process all findings). Values: "critical" | "important" | "minor". Severity hierarchy: critical > important > minor. Only findings at MIN_SEVERITY or above are processed; the rest are skipped silently.

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide the PRD path."
- **Path resolution (MUST do first):** `PRD_PATH` may be relative. Resolve it to an absolute path before ANY tool call:
  1. If `PRD_PATH` is already absolute (starts with `/`) → use as-is.
  2. If relative → resolve against the **current working directory** (the directory Claude Code was launched from). Use `Glob` with the relative pattern to find the actual file. If Glob finds exactly one match, use that absolute path. If zero matches, try common prefixes (`~/`, `./`) before reporting "not found".
  3. **Update `PRD_PATH`** to the resolved absolute path for all subsequent steps.
- If no findings are available (neither in context nor at `FINDINGS_SOURCE`) → STOP with "No findings found. Run the Review workflow first."

### Medium-Aware Mindset (CRITICAL)

Before proposing ANY fix, run this checklist mentally. **Do not default to adding or editing prose.**

<medium-check>
1. **What vs Why?** Is the finding about a contract/definition ("what") or about a decision/rationale ("why")?
   - "What" → belongs in code (types, schemas, state machine, validation, config)
   - "Why" → belongs in prose (design doc)

2. **Does the right medium already exist?** Check if the information is already captured in code files (e.g., `types.ts`, `state-machine.ts`, `errors.ts`). If yes:
   - The prose fix is: **reference the code**, don't duplicate it
   - The code fix is: update the code if it's wrong/incomplete

3. **Is the finding about scattered definitions?** If the same concept is defined in multiple prose sections:
   - The fix is consolidation to ONE source of truth (usually code), not "add another copy"
   - Prose sections should reference, not redefine

4. **Default direction:** Remove or simplify prose, don't add more. If the finding says "X is missing from section Y," first ask: should X be in prose at all, or should Y just reference the code where X is properly defined?

5. **Anti-pattern check:** Would the proposed fix create a new entry in the MediumGuide anti-patterns table? (e.g., TypeScript interface in markdown, state transitions in prose, same enum in 3 places)

6. **Verify, don't assume:** When step 2 concludes "the code already has it" — verify the SPECIFIC values/rules, not just that a relevant function exists. A function existing doesn't mean all related business rules are in code.
   - Are there magic numbers in prose that should be constants in code?
   - Are there conditions/rules described in prose without a corresponding function?
   - If yes → the finding TRANSFORMS: not "add prose docs" but "extract business rule to code"
</medium-check>

**Reference:** See [MediumGuide](../references/MediumGuide.md) for the decision matrix.

### Pipeline Pattern

The goal is to minimize user idle time:
- While the user reads finding N's context and recommendation, commit finding N-1 in the background
- The user makes a decision on N, you apply the fix, then immediately present N+1
- For the first finding: no previous commit, just present
- For the last finding: commit after user decides

### Presenting a Finding

For each finding, provide TWO things:
1. **Context** — read the relevant PRD/code sections so the user understands the actual state
2. **Recommendation** — your medium-aware proposal (which may differ from the original review finding's recommendation if the medium-check suggests a different approach)

If your recommendation differs from the original finding, briefly explain why (e.g., "The review suggested adding a transition table to the PRD, but the code already has it — the fix is to update prose references instead.").

### User Decisions (via AskUserQuestion)

After presenting, use the `AskUserQuestion` tool to collect the user's decision. This is faster and more structured than free-text input.

```
AskUserQuestion:
  question: "Finding #N — <Title> [Severity]"
  header: "Decision"
  options:
    - label: "Apply"
      description: "Execute the recommendation as presented"
    - label: "Skip"
      description: "No change for this finding, move to next"
    - label: "Stop"
      description: "Stop processing, defer remaining findings"
  multiSelect: false
```

The user can also select "Other" (always available) to:
- **Modify** — provide adjusted direction, then apply
- **Ask a question** — get clarification, then decide again

## Workflow

### 1. Load Findings

1. Collect the ordered finding list from `FINDINGS_SOURCE` or conversation context.
2. Extract: finding number, title, severity, original recommendation.
3. Filter by `MIN_SEVERITY`: drop findings below the threshold. Severity hierarchy: critical > important > minor.
   - `--min-severity=critical` → only Critical findings
   - `--min-severity=important` → Critical + Important
   - `--min-severity=minor` (default) → all findings
4. Report to user:
   ```
   Processing N findings from review of: <PRD_PATH>
   Min severity: <MIN_SEVERITY> (filtering out <skipped count> lower-severity findings)
   Critical: X | Important: Y | Minor: Z
   Starting with finding #1...
   ```

### 2. Process Findings (pipeline loop)

For each finding in severity order (Critical first):

<finding-pipeline-loop>

**Step A — Background commit (skip for first finding):**
If there is a completed fix from the previous finding, commit it as a background Task agent:
- Use `Task` tool with `subagent_type: "Bash"` and `run_in_background: true`
- The commit message follows project conventions (reference finding number and title)
- Do NOT wait for the commit to complete before presenting the next finding

**Step B — Present finding N:**

> **⚠️ CRITICAL — PRESENT, DON'T DECIDE:**
> Your job is to present the finding with context and a recommendation, then STOP and wait for the user's decision. Do NOT verify the finding yourself and declare a "verdict" (e.g., "FALSE POSITIVE", "not a real issue"). Do NOT skip findings based on your own assessment. Even if you believe a finding is wrong, present it with your analysis and let the user decide (apply/skip). The common failure mode is: you read the code, realize the agent was wrong, and write "Verdict: false positive — moving on." This bypasses the user's decision authority.

1. Read the relevant PRD sections and/or code files referenced by the finding.
2. Run the medium-check (see Instructions) to determine the right fix approach.
3. Present to the user:

```
---
## Finding #N — <Title> [Severity]

### Context
<What the PRD/code currently says — quote the relevant lines>

### Issue
<What's wrong and why it matters>

### Recommendation
<Your medium-aware fix proposal. State which files to change and how.>
<If this differs from the original review recommendation, explain why.>
---
```

**Step C — Collect user decision (AskUserQuestion):**
Use `AskUserQuestion` as described in "User Decisions" above. Then:
- If **Apply**: execute the fix (Edit/Write the files)
- If **Skip**: note it, move to next finding
- If **Stop**: report remaining findings count, exit the loop
- If **Other** (free text): interpret as modify direction or question. If question → answer, then ask again. If direction → adjust and execute.

</finding-pipeline-loop>

### 3. Background Commit Check

After the loop completes (or user stops), check the last background commit:
- Read the background task output
- If it succeeded: report commit hash
- If it failed: report the error

If there is an uncommitted fix from the final finding, commit it now (foreground).

### 4. Wrap Up

Report to user:

```
## Processing Complete

Findings processed: X / N
- Applied: A
- Skipped: S
- Remaining: R (deferred)

Commits: C created

<If remaining > 0:>
Run `/CraftPRD process-findings <PRD_PATH>` again to continue with remaining findings.

<If all applied:>
Consider running `/CraftPRD review <PRD_PATH>` to verify the fixes.
```

## STOP

**Do NOT continue past this point.** This workflow processes findings interactively — it does not auto-apply all findings.
