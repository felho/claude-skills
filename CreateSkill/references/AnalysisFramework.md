# Session Analysis Framework

This framework guides the Opus subagent when analyzing a session against a workflow.

## Analysis Dimensions

### 1. Workflow Step Fidelity

For each planned workflow step, determine:
- **Was it executed?** (yes / partially / skipped)
- **Was it executed in order?** (yes / reordered / merged with another step)
- **Did it succeed on first attempt?** (yes / needed retry / failed)
- **Were there deviations?** What did the agent actually do vs what was planned?

### 2. Pain Points & Inefficiencies

Look for these patterns in the session:

| Pattern | Signal in Session |
|---------|-------------------|
| **Wrong output path/format** | User asks to move/rename files after generation |
| **Missing convention** | User manually corrects formatting, naming, structure |
| **Sequential where parallel possible** | Independent tool calls made in separate turns |
| **Repeated attempts** | Same tool called multiple times with different args |
| **User correction loop** | User says "no, actually...", "change this to...", "that's wrong" |
| **Missing validation** | Errors discovered only when user reviews output |
| **Over-reading** | Agent reads files that don't contribute to the output |
| **Under-reading** | Agent misses key files, leading to incomplete analysis |
| **Context waste** | Agent reads large files fully when only a section was needed |
| **Missing error handling** | Agent hits an error and doesn't recover gracefully |

### 3. What Worked Well

Also identify smooth execution — steps that ran as designed with no issues. These should NOT be changed.

### 4. Post-Workflow User Actions

Track everything the user asked AFTER the workflow completed. These are strong signals for missing workflow steps:
- File moves/renames → naming convention issue
- Format fixes → report template issue
- "Add this to the report" → analysis step missed something
- Manual edits to output → output structure issue

## Output Format

The subagent should produce a structured report:

```markdown
## Session Overview
- What was the user doing?
- What workflow was invoked?
- How long did it take (turn count)?

## Step-by-Step Delta

| Step | Planned | Actual | Status |
|------|---------|--------|--------|
| 1 | [from workflow] | [what happened] | ok / deviated / skipped |

## Pain Points

### PP-1: [Title]
- **Category:** [from table above]
- **What happened:** [specific description with turn references]
- **Impact:** [how many extra turns / user interruptions this caused]
- **Suggested fix:** [concrete change to the workflow]

## What Worked Well
- [item]

## Improvement Proposals

### P0 (Critical)
1. [proposal] — effort: [line count estimate]

### P1 (Important)
1. [proposal] — effort: [line count estimate]

### P2 (Nice-to-have)
1. [proposal] — effort: [line count estimate]
```

## Session JSONL Reading Guide

Session files are JSONL. Each line is a JSON object:
- `type: "human"` → user message, look at `message.content`
- `type: "assistant"` → agent response, look at `message.content` array (text blocks + tool_use blocks)

**Tips for the subagent:**
- User corrections often appear in short human messages after long assistant outputs
- Tool call sequences reveal the actual execution order
- Look for `tool_use` blocks with `name: "Edit"` or `name: "Write"` — these show what files were modified
- Count turns to estimate efficiency
- The session may include post-workflow actions (fixes, commits) — analyze these separately
