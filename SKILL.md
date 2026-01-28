---
name: WritePrompt
description: Create, modify, and review agentic prompts/slash commands. USE WHEN create prompt OR new command OR slash command OR modify prompt OR update workflow OR improve prompt OR edit prompt OR prompt structure OR workflow prompt OR skill workflow OR add workflow to skill OR workflow file OR write workflow OR review prompt. Guides through 7 prompt levels with proper sections.
---

# WritePrompt

Framework for creating, modifying, and reviewing agentic prompts following the 7-level prompt hierarchy.

## Core Principle

> **The prompt is THE fundamental unit of agentic engineering.**

Every prompt should have the right level of complexity for its use case — no more, no less.

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **Write** | "create prompt", "new command", "modify prompt", "update workflow", "edit prompt", "improve prompt" | `Workflows/Write.md` |
| **Review** | "review this prompt", "check prompt quality", "audit prompt" | `Workflows/Review.md` |

## The 7 Prompt Levels

| Level | Name | Key Feature | When to Use |
|-------|------|-------------|-------------|
| 1 | High Level | Static, reusable | Simple one-shot tasks |
| 2 | Workflow | Sequential steps with I/O | Multi-step processes |
| 3 | Control Flow | Conditions and loops | Dynamic branching logic |
| 4 | Delegate | Spawns subagents | Parallel or distributed work |
| 5 | Higher Order | Accepts prompt files as input | Composable prompt systems |
| 6 | Template Metaprompt | Creates new prompts | Prompt generation |
| 7 | Self-Improving | Updates its own expertise | Learning systems |

## Quick Section Reference

**Essential sections** (most prompts need these):
- `Metadata` — YAML frontmatter (allowed-tools, description, argument-hint, model)
- `# Title` — Clear, action-oriented name
- `## Purpose` — What it does and when to use it
- `## Variables` — Dynamic ($1, $ARGUMENTS) and static values
- `## Workflow` — Numbered steps to execute
- `## Report` — How to present results

**Advanced sections** (for complex prompts):
- `## Instructions` — Rules, constraints, edge cases
- `## Relevant Files` — Files to read/modify
- `## Codebase Structure` — Directory layout context
- `## Expertise` — Accumulated domain knowledge (Level 7)
- `## Template` — Output format for metaprompts (Level 6)
- `## Examples` — Usage demonstrations

## Examples

**Example 1: Create a new slash command**
```
User: "Create a command to summarize git history"
→ Invokes Write workflow (create mode)
→ Asks clarifying questions (output format? date range?)
→ Determines Level 2 (Workflow) is appropriate
→ Creates .claude/commands/git-summary.md with proper sections
```

**Example 2: Modify existing prompt**
```
User: "Add a verification step to the Check workflow"
→ Invokes Write workflow (modify mode)
→ Reads existing prompt, identifies what needs to change
→ Adds the new step, renumbers subsequent steps
→ Updates the prompt file
```

**Example 3: Review prompt quality**
```
User: "Review this prompt and suggest improvements"
→ Invokes Review workflow
→ Identifies current level and sections
→ Suggests missing sections or level upgrade
→ Provides concrete recommendations (does NOT modify)
```

**Example 4: Complex automation prompt**
```
User: "Create a command that processes multiple files in parallel"
→ Invokes Write workflow (create mode)
→ Recognizes need for Level 4 (Delegate)
→ Includes Task tool delegation pattern
→ Adds proper agent configuration variables
```

## References

- [PromptLevels](references/PromptLevels.md) — Detailed breakdown of all 7 levels with examples
- [PromptSections](references/PromptSections.md) — Complete section reference with usage guidelines
