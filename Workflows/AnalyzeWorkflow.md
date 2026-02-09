# Analyze Workflow Usage

Analyze a Claude Code session against a workflow definition to identify improvement opportunities.

Execute the `Workflow` section and present findings following the `Report` format.

## Variables

WORKFLOW_PATH: Path to the workflow .md file (e.g., skills/LearnFromVideo/Workflows/Analyze.md)
SESSION_ID: Claude Code session ID (e.g., 822bc6ff-cff8-414a-ba31-46141a54dbbb)
PROJECT_PATH: (optional) Project path override — defaults to current project

## Instructions

- If `WORKFLOW_PATH` is empty -> STOP and report: `"Usage: Provide workflow path and session ID"`
- If `SESSION_ID` is empty -> STOP and report: `"Usage: Provide workflow path and session ID"`
- The session analysis MUST be delegated to an Opus subagent to protect the main context from large JSONL files
- All analysis output should be in English regardless of conversation language
- Do NOT modify any files during analysis — this workflow is read-only
- When multiple sessions are provided (comma-separated), analyze each and look for patterns across runs

## Workflow

### 1. Validate Inputs

- If `WORKFLOW_PATH` is empty -> STOP and report: `"Usage: Provide workflow path and session ID"`
- If `SESSION_ID` is empty -> STOP and report: `"Usage: Provide workflow path and session ID"`
- Verify workflow file exists: Read `WORKFLOW_PATH`
- Locate session file: Glob for `**/*{SESSION_ID}*.jsonl` under `~/.claude/projects/`
- If session file not found -> STOP and report: `"Error: Session not found: {SESSION_ID}"`

### 2. Load Workflow Context

- Read the workflow file at `WORKFLOW_PATH`
- Extract the list of planned steps (numbered headings under `## Workflow`)
- Read the analysis framework: `references/AnalysisFramework.md`

### 3. Delegate Session Analysis to Subagent

Launch an Opus Task subagent with the following context:

<subagent-prompt>
You are analyzing a Claude Code session to identify how well a planned workflow was followed, and what improvements could be made.

**Planned Workflow:**
{paste the full workflow file content here}

**Analysis Framework:**
{paste the AnalysisFramework.md content here}

**Session file:** {session file path}

**Your task:**
1. Read the FULL session JSONL file
2. Map each user message and assistant action to the planned workflow steps
3. Identify the delta between planned and actual execution
4. Identify pain points using the framework categories
5. Identify what worked well (do NOT suggest changing these)
6. Track post-workflow user actions (corrections, fixes, additions)
7. Propose concrete improvements with priority levels (P0/P1/P2)

**Output:** Follow the exact output format from the Analysis Framework.

Be thorough — read the entire session. This analysis will be used to improve the workflow.
</subagent-prompt>

- Use `model: opus` for the subagent
- Wait for the subagent to complete

### 4. Present Findings

- Display the subagent's analysis to the user
- Highlight the P0 (critical) improvements prominently
- Ask: "Which improvements would you like to apply?"

## Error Messages (Use Exactly)

- Missing input: `"Usage: Provide workflow path and session ID. Example: optimize skills/X/Workflows/Y.md based on session abc123"`
- Workflow not found: `"Error: Workflow file not found: {path}"`
- Session not found: `"Error: Session not found: {session_id}. Use search-session to find the correct ID."`

## Report

After the subagent completes, present:

```
Workflow: {workflow-path}
Session: {session-id} ({turn-count} turns)

## Pain Points Found: {count}
{P0 items listed first}

## What Worked Well: {count} items

## Improvement Proposals
P0 (Critical): {count}
P1 (Important): {count}
P2 (Nice-to-have): {count}

Which improvements would you like to apply?
```
