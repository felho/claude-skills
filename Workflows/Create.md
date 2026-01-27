# Create Prompt Workflow

Guide for creating new agentic prompts/slash commands.

## Input

`USER_REQUEST`: The user's description of what they want the prompt to do.

## Workflow

### 1. Understand Requirements

Analyze the user request to determine:
- **Primary purpose**: What should the prompt accomplish?
- **Input needs**: What variables/arguments are required?
- **Output format**: How should results be presented?
- **Tool requirements**: Which tools are needed?

### 2. Ask Clarifying Questions (if needed)

If the request is ambiguous, ask about:
- Expected inputs and their format
- Desired output structure
- Error handling preferences
- Whether it needs to delegate to agents
- File paths or directories involved

**Keep questions focused** — ask only what's necessary to determine the right level and structure.

### 3. Determine Prompt Level

Based on requirements, select the appropriate level:

| If the prompt needs to... | Use Level |
|---------------------------|-----------|
| Execute a simple static task | 1 - High Level |
| Follow sequential steps with I/O | 2 - Workflow |
| Use conditions or loops | 3 - Control Flow |
| Spawn subagents for parallel work | 4 - Delegate |
| Accept another prompt file as input | 5 - Higher Order |
| Generate new prompts | 6 - Template Metaprompt |
| Accumulate knowledge over time | 7 - Self-Improving |

**Default to Level 2** for most use cases. Only go higher when truly needed.

### 4. Select Required Sections

Based on the level, include appropriate sections:

**Always include:**
- Metadata (YAML frontmatter)
- Title
- Workflow (except Level 1)

**Include when relevant:**
- Variables (if any input needed)
- Instructions (if rules/constraints exist)
- Report (if specific output format needed)
- Relevant Files (if working with specific files)
- Codebase Structure (if directory context helps)

**Level-specific:**
- Level 4: Agent configuration in Variables
- Level 6: Template section with output format
- Level 7: Expertise section for accumulated knowledge

### 5. Draft the Prompt

Create the prompt following this structure:

```markdown
---
description: <one-line description>
argument-hint: [<arg1>] [<arg2>]
allowed-tools: <comma-separated tools>
model: <sonnet|opus if needed>
---

# <Prompt Title>

<Brief purpose statement referencing Workflow and Report sections>

## Variables

<DYNAMIC_VAR>: $1
<STATIC_VAR>: <fixed value>

## Instructions

- <Rule or constraint>
- <Edge case handling>
- <Important behavior>

## Workflow

1. <First step>
2. <Second step>
   <loop-block if needed>
   - Loop step
   </loop-block>
3. <Final step>

## Report

<Output format specification>
```

### 6. Validate the Prompt

Check that:
- [ ] Variables have clear names (SCREAMING_CASE)
- [ ] Workflow steps are numbered and clear
- [ ] STOP conditions are explicit for missing required input
- [ ] Tool usage matches `allowed-tools` in metadata
- [ ] Report format is specified if outputs matter

### 7. Save and Confirm

- Save to `.claude/commands/<prompt-name>.md`
- Use kebab-case for filename
- Confirm creation with summary of what was built

## Output

Provide a summary:

```
✅ Prompt Created

File: .claude/commands/<name>.md
Level: <N> - <Level Name>
Purpose: <one-line description>

Sections included:
- <section 1>
- <section 2>
- ...

Usage: /<command-name> <arguments>
```
