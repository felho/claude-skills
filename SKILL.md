---
name: CreateSkill
description: Create and validate Claude Code skills. USE WHEN create skill, new skill, skill structure, validate skill, audit skill, canonicalize skill.
---

# CreateSkill

Skill creation framework following PAI conventions with cherry-picked additions.

## Core Specification

**Follow:** [pai/CreateSkill.md](references/pai/CreateSkill.md) → [pai/SkillSystem.md](references/pai/SkillSystem.md)

## Additions

### 1. Test with Multiple Models

Always test skills with Haiku, Sonnet, AND Opus before finalizing. Different models may interpret instructions differently.

### 2. Anti-Patterns to Avoid

- **Vague descriptions** — Be specific with USE WHEN trigger keywords
- **Deep nesting** — Keep references one level from SKILL.md
- **XML tags in body** — Use standard markdown headings

### 3. Progressive Disclosure

Keep SKILL.md under 500 lines. Use `references/` for detailed docs.

## Examples

**Example 1: Create a new skill**
```
User: "Create a skill for managing invoices"
→ Read pai/CreateSkill.md and pai/SkillSystem.md
→ Create .claude/skills/InvoiceManager/SKILL.md
→ Use TitleCase naming throughout
→ Include USE WHEN in description
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
