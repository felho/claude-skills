# CreateSkill

A Claude Code skill for creating, validating, and optimizing Claude Code skills. It follows the PAI (Personal AI Infrastructure) conventions with cherry-picked additions, and includes workflow optimization through session analysis.

## Why This Skill Exists

Claude Code skills are the building blocks of an agentic workflow system. Without a standardized creation process, skills end up with inconsistent naming, missing trigger keywords, unclear structure, and no examples. CreateSkill enforces a consistent structure so that every skill is discoverable, well-documented, and follows established conventions.

## How It Works

CreateSkill serves two purposes:

1. **Skill creation and validation** — Ensures new skills follow the PAI specification (TitleCase naming, USE WHEN triggers, Workflow Routing tables, Examples sections)
2. **Workflow optimization** — Analyzes real Claude Code sessions against workflow definitions to find improvement opportunities

### The PAI Specification

Every skill must follow the structure defined in `references/pai/SkillSystem.md`:

```
SkillName/                          # TitleCase directory name
├── SKILL.md                        # Main skill file (YAML frontmatter + markdown body)
├── Workflows/                      # Work execution workflows (TitleCase filenames)
│   └── WorkflowName.md
├── references/                     # Supporting documentation
│   └── DetailedGuide.md
└── Tools/                          # CLI tools (always present, even if empty)
    └── .gitkeep
```

### SKILL.md Requirements

```yaml
---
name: SkillName                     # TitleCase, mandatory
description: What it does. USE WHEN trigger keywords. # Single line, USE WHEN mandatory
---
```

The markdown body must include:
- `## Workflow Routing` — Table mapping triggers to workflow files
- `## Examples` — 2-3 concrete usage patterns showing user input and expected behavior

## Naming Conventions

### Verb+Noun Pattern

Skill names describe what you DO, not what the artifact IS:

| Good | Bad | Why |
|------|-----|-----|
| `UseGit` | `GitWorkflow` | Describes the action |
| `WriteCode` | `TDDWorkflow` | TDD is one aspect of writing code |
| `CreateSkill` | `SkillCreator` | Action-oriented |
| `DevelopCLI` | `CLIWorkflow` | Verb+Noun is clearer |

### No "Workflow" Suffix

Since every skill already has a `Workflows/` directory, adding "Workflow" to the name is redundant:

```
Bad:  GitWorkflow/Workflows/PreCommit.md  →  "Git Workflow's Workflows"
Good: UseGit/Workflows/PreCommit.md       →  "UseGit's Workflows"
```

### Merge vs Create Decision

- **Merge** into an existing skill when content is closely related and shares trigger patterns
- **Create** a new skill when the domain is distinct and has its own triggers

Ask: "Would a user expect to find this in skill X, or would they search for it separately?"

## Workflows

### AnalyzeWorkflow

Analyzes a Claude Code session against a workflow definition to identify improvement opportunities.

```
User: "Optimize the LearnFromVideo Analyze workflow based on session 822bc6ff"
→ Reads the workflow definition
→ Launches an Opus subagent to analyze the session JSONL
→ Maps each user message and agent action to planned workflow steps
→ Identifies: what worked well, pain points, deviations from the plan
→ Presents improvement proposals prioritized as P0/P1/P2
```

The analysis uses a structured framework (`references/AnalysisFramework.md`) that looks for patterns like:
- Wrong output path/format (user had to move/rename files)
- Missing conventions (user manually corrected formatting)
- Sequential operations where parallel was possible
- Repeated attempts with different arguments
- User correction loops ("no, actually...")

Supports analyzing multiple sessions (comma-separated IDs) to find recurring patterns across runs.

### ApplyAnalysis

Applies user-approved improvements from the AnalyzeWorkflow to the actual workflow file:

```
User: "Apply improvements 1, 3, and 5"
→ Reads current workflow file
→ Plans edits for each approved improvement
→ Applies in priority order (P0 first)
→ Verifies step numbering, references, and logical flow
→ Presents summary of changes
```

## Companion Skill: WritePrompt

When creating workflow files within a skill, CreateSkill should be used together with WritePrompt:

| CreateSkill handles | WritePrompt handles |
|---------------------|---------------------|
| Skill structure validation | Prompt content quality |
| Directory layout | Section completeness |
| Naming conventions | Variable naming (SCREAMING_CASE) |
| SKILL.md format | Workflow step clarity |

The recommended sequence: invoke CreateSkill first (for structure), then WritePrompt (for prompt quality).

## Directory Structure

```
CreateSkill/
├── SKILL.md                                # Main skill definition
├── README.md                               # This file
├── Workflows/
│   ├── AnalyzeWorkflow.md                  # Session analysis workflow
│   └── ApplyAnalysis.md                    # Apply improvements workflow
├── references/
│   ├── AnalysisFramework.md                # Framework for session analysis
│   ├── NamingConventions.md                # Verb+Noun, merge vs create
│   └── pai/                                # PAI specification (authoritative)
│       ├── CreateSkill.md                  # Skill creation guide
│       └── SkillSystem.md                  # Complete skill structure spec
├── resources/                              # Source materials (historical reference)
│   ├── AnthropicSkillCreator/              # Official Anthropic skill-creator
│   └── CompoundCreateAgentSkills/          # Compound Engineering plugin
└── Tools/
    └── .gitkeep
```

### About the `resources/` Directory

The `resources/` folder contains the original source materials that influenced this skill's design:

- **AnthropicSkillCreator** — The official Anthropic skill creator from [anthropics/skills](https://github.com/anthropics/skills). Notable concepts: "context window is a public good," degrees of freedom, Assets vs References.
- **CompoundCreateAgentSkills** — From the [Compound Engineering plugin](https://github.com/EveryInc/every-marketplace). Notable concepts: "Skills Are Prompts," anti-patterns list, multi-model testing.

These are kept for reference and cherry-picking, not for Claude to read during skill creation.

## Usage Examples

### Example 1: Create a new skill

```
User: "Create a skill for managing invoices"
→ Reads PAI specification (SkillSystem.md)
→ Creates ~/.claude/skills/ManageInvoices/SKILL.md
→ Uses Verb+Noun naming, includes USE WHEN triggers
→ Creates Workflows/, references/, Tools/ directories
→ Adds Examples section with 2-3 usage patterns
```

### Example 2: Validate an existing skill

```
User: "Check if my skill is valid"
→ Checks against SkillSystem.md checklist:
  - TitleCase naming? ✓
  - USE WHEN in description? ✓
  - Workflow Routing table? ✗ Missing
  - Examples section? ✗ Missing
→ Reports issues and suggests fixes
```

### Example 3: Optimize a workflow from session data

```
User: "Optimize the CraftPRD Review workflow based on sessions abc and def"
→ Analyzes both sessions against the Review workflow
→ Finds: 3 recurring issues, 2 one-off deviations
→ Proposes: 2 P0 improvements (recurring), 1 P1, 2 P2
→ User approves P0 and P1 → applies edits to workflow file
```

## Skill Validation Checklist

When creating or auditing a skill, verify:

- [ ] Skill directory uses TitleCase
- [ ] YAML `name:` uses TitleCase
- [ ] Single-line `description` with `USE WHEN` clause
- [ ] `## Workflow Routing` section with table format
- [ ] `## Examples` section with 2-3 usage patterns
- [ ] `Tools/` directory exists (even if empty)
- [ ] All workflow files use TitleCase
- [ ] SKILL.md is under 500 lines (use `references/` for details)

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| TitleCase naming | Following PAI convention for consistency |
| USE WHEN mandatory | Explicit trigger keywords improve skill discovery |
| Thin SKILL.md | Only additions, delegate to PAI spec |
| resources/ separate | Historical context shouldn't pollute Claude's decisions |
| Companion WritePrompt | Workflow files are prompts — they need prompt engineering rigor |

## Installation

This skill is part of the `~/.claude/skills/` directory and activates automatically when Claude Code detects skill-creation or workflow-optimization requests. No additional setup is required.

## When Does It Activate?

CreateSkill triggers on:
- "skill" + intent (create, make, build, want, need)
- "workflow" + intent (create, make, add, new)
- "automate [task]"
- Skill structure, validate, audit mentions
- "optimize workflow", "analyze session", "workflow delta"
