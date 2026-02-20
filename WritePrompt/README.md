# WritePrompt

A Claude Code skill for creating, reviewing, and hardening agentic prompts and slash commands. It provides a structured framework based on a 7-level prompt hierarchy, ensuring prompts are well-organized, appropriately complex, and include proper validation hooks.

## Why This Skill Exists

Prompts are the fundamental unit of agentic engineering. A poorly structured prompt leads to inconsistent results, missed edge cases, and wasted context window. WritePrompt brings engineering rigor to prompt creation — the same way you'd design an API contract or a database schema, but for natural language instructions that agents follow.

## The 7-Level Prompt Hierarchy

WritePrompt classifies every prompt into one of 7 levels based on its complexity and capabilities:

| Level | Name | Key Feature | Example |
|-------|------|-------------|---------|
| **L1** | High Level | Static, no dynamic input | "Summarize this file" |
| **L2** | Workflow | Sequential steps with I/O | Multi-step analysis pipeline |
| **L3** | Control Flow | Conditions, loops, branching | Processing items with skip logic |
| **L4** | Delegate | Spawns parallel subagents | Review with multiple expert agents |
| **L5** | Higher Order | Accepts another prompt as input | "Run this prompt against all files" |
| **L6** | Template Metaprompt | Generates new prompts | Prompt factory for new domains |
| **L7** | Self-Improving | Updates itself with learned knowledge | Expert system that accumulates domain expertise |

Most practical prompts fall in the L2-L4 range. The level determines which sections are required and what patterns to use.

## How It Works

WritePrompt has three workflows:

### 1. Write (`/WritePrompt` or "create prompt")

Creates a new prompt or modifies an existing one. The workflow:

1. Determines whether to create or modify
2. Understands the requirements (purpose, inputs, outputs, tools)
3. Asks clarifying questions if needed
4. Selects the appropriate prompt level (L1-L7)
5. Selects required sections for that level
6. Writes the prompt with proper structure
7. Validates the result against a checklist
8. Saves the file

### 2. Review (`/WritePrompt review`)

Analyzes an existing prompt for structural issues:

1. Reads and parses the prompt (YAML frontmatter + sections)
2. Identifies the current level
3. Checks required sections are present for that level
4. Validates content is in the correct sections (semantic placement)
5. Audits syntax (metadata, variables, workflow steps)
6. Validates control flow patterns (STOP conditions, loops, named blocks)
7. Reports issues by severity (High/Medium/Low)
8. Suggests a level upgrade path if applicable

### 3. Harden (`/WritePrompt harden`)

Adds self-validation hooks to a prompt:

1. Loads the SelfValidationHooks reference
2. Analyzes the target prompt's tool usage and outputs
3. Recommends a hook strategy (PreToolUse, PostToolUse, Stop, or combined)
4. Designs validator logic with pass/fail conditions
5. Writes validator scripts (Python with PEP 723 format)
6. Adds hooks to the prompt's YAML frontmatter
7. Verifies the setup works

## Prompt Structure

Every prompt created by WritePrompt follows this structure:

```yaml
---
name: PromptName
description: What it does. USE WHEN trigger conditions.
argument-hint: <required-arg> [optional-arg]
allowed-tools: Read, Write, Bash, Grep, Glob
model: opus
---
```

```markdown
# Prompt Title

## Purpose
What it does and when to use it.

## Variables
SCREAMING_CASE variable definitions with defaults.

## Instructions
Bullet-point rules, constraints, and guardrails.

## Workflow
Numbered steps with conditional logic and loop blocks.

## Report
How to present results after execution.
```

### Section Requirements by Level

| Section | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
|---------|----|----|----|----|----|----|-----|
| Purpose | Required | Required | Required | Required | Required | Required | Required |
| Variables | - | Required | Required | Required | Required | Required | Required |
| Instructions | Required | Required | Required | Required | Required | Required | Required |
| Workflow | - | Required | Required | Required | Required | Optional | Required |
| Report | Optional | Required | Required | Required | Optional | Optional | Required |
| Template | - | - | - | - | - | Required | - |
| Expertise | - | - | - | - | - | - | Required |

## Self-Validation Hooks

The Harden workflow adds automated validation to prompts using Claude Code's hook system:

- **PreToolUse hooks** — Block dangerous operations before they execute (e.g., prevent writing to wrong directories)
- **PostToolUse hooks** — Validate individual file operations after they complete (e.g., check file format)
- **Stop hooks** — Run final/global validation when the agent finishes (e.g., verify all required sections exist in output)

Validators are Python scripts using the PEP 723 format (`uv run --script`) with structured JSON output for pass/block decisions.

## Directory Structure

```
WritePrompt/
├── SKILL.md                            # Main skill definition with 7-level hierarchy
├── README.md                           # This file
├── Workflows/
│   ├── Write.md                        # Create/modify prompts (8 steps)
│   ├── Review.md                       # Audit existing prompts (9 steps)
│   └── Harden.md                       # Add validation hooks (8 steps)
├── references/
│   ├── PromptLevels.md                 # Detailed breakdown of all 7 levels
│   ├── PromptSections.md              # Complete section reference and syntax
│   └── SelfValidationHooks.md         # Hook types, validator architecture, patterns
└── Tools/
    └── .gitkeep
```

## Usage Examples

### Example 1: Create a new workflow prompt

```
User: "Create a prompt for analyzing code coverage reports"
→ WritePrompt determines this is a new L2 (Workflow) prompt
→ Asks: "What format are the reports? What's the output?"
→ Creates prompt with Variables, Instructions, Workflow, Report sections
→ Validates: all variables referenced, STOP conditions present, tools specified
```

### Example 2: Review an existing prompt

```
User: "Review the CraftPRD Review workflow"
→ Reads the prompt, identifies it as L4 (Delegate — spawns subagents)
→ Checks: required sections present, control flow syntax correct
→ Finds: missing STOP condition in loop block, variable not in SCREAMING_CASE
→ Reports issues with severity and suggests fixes
```

### Example 3: Add validation to a prompt

```
User: "Harden the LearnFromVideo Analyze workflow"
→ Analyzes: prompt writes reports to MEMORY directory, captures frames
→ Recommends: PostToolUse hook for Write (validate report format) + Stop hook (verify all sections)
→ Creates validator scripts with PEP 723 format
→ Adds hooks to YAML frontmatter
```

### Example 4: Upgrade a simple prompt to a higher level

```
User: "My summarize prompt needs to handle multiple files"
→ Reviews current prompt: L1 (static, single file)
→ Suggests upgrade to L3 (control flow with loop over files)
→ Adds Variables section, loop block in Workflow, file count in Report
```

## Key Design Patterns

### STOP Conditions

Every workflow must define when to stop and what error to show:

```markdown
- If `VIDEO_URL` is empty → STOP and report: "Usage: Provide a YouTube URL"
```

### Loop Blocks

Named with `-loop` suffix for clarity:

```markdown
<process-files-loop>
- Read next file
- Extract patterns
- If file is empty → skip
- Add to results
</process-files-loop>
```

### Variable Naming

Always SCREAMING_CASE with clear defaults:

```markdown
## Variables
REPORT_DIR: ~/.claude/MEMORY/LEARNINGS
MAX_FILES: 10
FOCUS_AREA: (optional, extracted from user message)
```

## Installation

This skill is part of the `~/.claude/skills/` directory and activates automatically when Claude Code detects prompt-related requests. No additional setup is required.

## When Does It Activate?

WritePrompt triggers on phrases like:
- "create prompt", "new command", "slash command"
- "modify prompt", "update workflow", "improve prompt"
- "review prompt", "harden prompt", "add validation"
- "skill workflow", "add workflow to skill"
