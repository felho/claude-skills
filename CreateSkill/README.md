# CreateSkill - History and Design Decisions

This document explains the story behind this skill's layered architecture and contains TODOs for future improvements.

## Why This Structure?

We wanted a skill creation framework that:
1. **Follows PAI conventions** — TitleCase naming, USE WHEN triggers, Workflow Routing
2. **Is upgrade-friendly** — Source materials can be updated independently
3. **Avoids noise** — Claude only reads what's needed, not historical context

**Why PAI?** Over time, we want to move closer and closer to the PAI (Personal AI Infrastructure) system. By adopting PAI conventions now, we ensure compatibility and easier migration as we integrate more PAI patterns into our workflow.

## Layered Architecture

```
CreateSkill/
├── SKILL.md                    ← Our thin layer (Claude reads this)
├── references/pai/             ← Active spec (Claude follows this)
│   ├── CreateSkill.md
│   └── SkillSystem.md
└── resources/                  ← Source materials (historical reference)
    ├── AnthropicSkillCreator/
    └── CompoundCreateAgentSkills/
```

**Flow:**
1. SKILL.md triggers → routes to `references/pai/CreateSkill.md`
2. That routes to `references/pai/SkillSystem.md` (the actual spec)
3. Our SKILL.md adds cherry-picked improvements on top

## Source Materials (resources/)

These are kept for reference and potential cherry-picking, not for Claude to read during skill creation.

### AnthropicSkillCreator

Official Anthropic skill-creator from [anthropics/skills](https://github.com/anthropics/skills).

**Interesting concepts:**
- "Context window is a public good"
- Degrees of freedom (high/medium/low based on task fragility)
- Assets vs References distinction
- Scripts: `init_skill.py`, `package_skill.py`, `quick_validate.py`

### CompoundCreateAgentSkills

From [compound-engineering plugin](https://github.com/EveryInc/every-marketplace).

**Interesting concepts:**
- "Skills Are Prompts"
- Gerund naming convention (verb + -ing)
- Anti-patterns list
- Test with Haiku, Sonnet, AND Opus

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| TitleCase naming | Following PAI convention for consistency |
| USE WHEN mandatory | Explicit trigger keywords improve skill discovery |
| Thin SKILL.md | Only additions, delegate to PAI spec |
| resources/ separate | Historical context shouldn't pollute Claude's decisions |

## TODOs

- [ ] **Test Anthropic's validation scripts** — Check if `quick_validate.py` works with our TitleCase/PAI structure or needs adaptation
- [ ] **Test Anthropic's init script** — Check if `init_skill.py` generates compatible structure or needs our own version
- [ ] **Consider custom validator** — May need to create our own that checks PAI-specific rules (USE WHEN, TitleCase, etc.)
