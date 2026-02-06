---
description: Iterate on a PRD adding technical detail while monitoring complexity signals
argument-hint: <prd-path> [<focus-area>]
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion
---

# Deepen — Iterate on PRD with Complexity Monitoring

Add technical detail to a PRD through iterative discussion. Continuously monitors complexity signals and warns when the PRD is outgrowing prose.

## Variables

PRD_PATH: $1
FOCUS_AREA: $2 (optional — specific section or topic to deepen)
COMPLEXITY_SIGNALS_REF: references/ComplexitySignals.md (relative to CraftPRD skill directory)

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide a path to the PRD file."
- This workflow is **iterative and interactive**. Each iteration deepens one area, then checks complexity.
- Write all PRD content in **English**.
- When adding technical detail, prefer prose descriptions over code blocks. If a code block feels necessary, that's a complexity signal.
- After each significant addition, run the complexity check. If signals fire, inform the user but don't stop — let them decide when to restructure.

## Workflow

### 1. Read PRD and Assess State

- Read the PRD at `PRD_PATH`.
  - If not found → STOP with "PRD not found: {PRD_PATH}"
- Read `COMPLEXITY_SIGNALS_REF` for the signal thresholds.
- Summarize the current state: line count, number of code blocks, cross-references.
- If `FOCUS_AREA` is provided, identify the relevant section(s).

### 2. Identify What to Deepen

- If `FOCUS_AREA` is specified → focus there.
- If not → ask the user: "Which area would you like to deepen?" and suggest the least-developed sections based on the current content.

### 3. Iterative Deepening

<deepen-loop>

**3a. Discuss the area with the user.** Ask targeted questions:
- What are the key technical decisions here?
- What are the edge cases or failure modes?
- Are there trade-offs to document?
- What should the behavior be when X goes wrong?

**3b. Update the PRD** with the discussed content. Edit the existing file — don't rewrite the whole thing.

**3c. Run complexity check.** Count:
- Total line count
- Number of TypeScript/code blocks
- Number of SQL/schema blocks
- Number of cross-references ("see section", "as described in")
- Repeated enum values across sections
- ASCII diagrams (state machine, sequence)

Compare against thresholds from `COMPLEXITY_SIGNALS_REF`.

**3d. Report complexity status.**

If any thresholds are exceeded:
```
Complexity signal: <signal name> (<current value> exceeds threshold of <threshold>)
```

If multiple signals fire:
```
Multiple complexity signals firing:
- <signal 1>: <value> (threshold: <threshold>)
- <signal 2>: <value> (threshold: <threshold>)

Consider running the Structure workflow to split this PRD into proper mediums.
The PRD isn't broken — it's outgrown prose. See ComplexitySignals.md for details.
```

**3e. Ask the user** if they want to:
- Continue deepening another area
- Run the Structure workflow
- Stop for now

If the user wants to continue → repeat from 3a with the next area.

</deepen-loop>

### 4. Final Summary

When the user stops iterating, provide a final complexity report.

## Report

```
Deepen session complete: <PRD_PATH>

Changes made:
- <area 1>: <what was added/changed>
- <area 2>: <what was added/changed>

Complexity status:
- Lines: <count> (<status>)
- Code blocks: <count> (<status>)
- Cross-references: <count> (<status>)

<If signals are firing:>
Recommendation: Run the Structure workflow to split technical contracts into code.

<If no signals:>
PRD is within complexity thresholds. Continue deepening or proceed to Review.
```
