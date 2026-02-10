---
name: CreateSkill
description: Create, validate, and optimize Claude Code skills. USE WHEN user says "skill" + intent (create, make, build, want, need, help me with) OR "workflow" + intent (create, make, add, new) OR "workflow for/to" OR "automate [task]" OR mentions skill structure, validate, audit OR optimize workflow OR improve workflow OR analyze session OR workflow delta.
---

# CreateSkill

Skill creation framework following PAI conventions with cherry-picked additions. Also includes workflow optimization via session analysis.

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **AnalyzeWorkflow** | "optimize workflow", "analyze session", "workflow delta" | `Workflows/AnalyzeWorkflow.md` |
| **ApplyAnalysis** | "apply improvements", "apply the changes" | `Workflows/ApplyAnalysis.md` |

## Core Specification

**Follow:** [pai/CreateSkill.md](references/pai/CreateSkill.md) → [pai/SkillSystem.md](references/pai/SkillSystem.md)

**Naming:** [NamingConventions.md](references/NamingConventions.md) — Verb+Noun pattern, merge vs create decisions

## Additions

### 1. Test with Multiple Models

Always test skills with Haiku, Sonnet, AND Opus before finalizing. Different models may interpret instructions differently.

### 2. Anti-Patterns to Avoid

- **Vague descriptions** — Be specific with USE WHEN trigger keywords
- **Deep nesting** — Keep references one level from SKILL.md
- **XML tags in body** — Use standard markdown headings

### 3. Progressive Disclosure

Keep SKILL.md under 500 lines. Use `references/` for detailed docs.

### 4. Companion Skill: WritePrompt

**When creating workflow files, also invoke WritePrompt skill.**

Workflow files are prompts — they benefit from WritePrompt's:
- 7-level prompt hierarchy (most workflows are Level 2-3)
- Section guidelines (Variables, Instructions, Workflow, Report)
- Prompt engineering best practices

| CreateSkill handles | WritePrompt handles |
|---------------------|---------------------|
| Skill structure validation | Prompt content quality |
| Directory layout | Section completeness |
| Naming conventions | Variable naming (SCREAMING_CASE) |
| SKILL.md format | Workflow step clarity |

## Examples

**Example 1: Create a new skill**
```
User: "Create a skill for managing invoices"
→ Read pai/CreateSkill.md, pai/SkillSystem.md, and ProjectConventions.md
→ Create .claude/skills/ManageInvoices/SKILL.md (Verb+Noun pattern)
→ Include USE WHEN in description
→ Add Workflows/, references/, Tools/ directories
→ Add Examples section
```

**Example 2: Validate existing skill**
```
User: "Check if my skill is valid"
→ Check against pai/SkillSystem.md checklist
→ Verify TitleCase naming
→ Ensure USE WHEN in description
→ Ensure Examples section exists
```

**Example 3: Optimize a workflow from session data**
```
User: "Optimize the LearnFromVideo Analyze workflow based on session 822bc6ff"
→ Invokes AnalyzeWorkflow workflow
→ Reads the workflow, launches Opus subagent to analyze session JSONL
→ Presents delta analysis: planned vs actual, pain points, proposals
→ User approves → invokes ApplyAnalysis to edit the workflow
```

**Example 4: Analyze multiple sessions for patterns**
```
User: "I've used the CraftPRD Review workflow in sessions abc and def, optimize it"
→ Invokes AnalyzeWorkflow with comma-separated session IDs
→ Finds recurring patterns across runs (repeated issues vs one-offs)
→ Prioritizes improvements that fix repeated pain points
```
