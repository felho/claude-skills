# Skill Naming Conventions

Extends [pai/SkillSystem.md](pai/SkillSystem.md) with conventions for naming and organizing skills.

## Naming: Verb+Noun Pattern

Skill names follow **Verb+Noun** pattern describing what you DO:

| ✅ Good | ❌ Bad | Why |
|---------|--------|-----|
| `UseGit` | `GitWorkflow` | Describes action, not artifact type |
| `WriteCode` | `TDDWorkflow` | TDD is just one aspect of writing code |
| `DevelopCLI` | `CLIWorkflow` | Verb+Noun is clearer |
| `DebugBrowser` | `BrowserDebug` | Consistent Verb+Noun order |
| `CreateSkill` | `SkillCreator` | Action-oriented |

**Rationale:** The name should answer "What do I do with this skill?" not "What type of artifact is this?"

## Why No "*Workflow" Suffix

**Don't add "Workflow" to skill names** — it's redundant.

Since every skill can have a `Workflows/` directory, the skill itself is a container for workflows. Adding "Workflow" to the name creates redundancy:

```
❌ GitWorkflow/Workflows/PreCommit.md → "Git Workflow's Workflows"
✅ UseGit/Workflows/PreCommit.md → "UseGit's Workflows"
```

## Merge vs Create Decision

**When to MERGE into existing skill (as reference):**
- Content is closely related to an existing skill's domain
- Same trigger patterns would apply
- Example: CodeQuality → WriteCode/references/ (both about "how to write code")

**When to CREATE new skill:**
- Distinct domain with its own trigger patterns
- Would confuse users if bundled with existing skill
- Has potential for its own workflows

**Ask:** "Would a user expect to find this in skill X, or would they search for it separately?"

## Why .gitkeep in Empty Directories

Every skill has the same directory structure, even if directories are empty:

```
SkillName/
├── SKILL.md
├── Workflows/       # .gitkeep if empty
├── references/      # .gitkeep if empty
└── Tools/           # .gitkeep if empty
```

**Why:**
1. **Consistency** — Same structure across all skills, easier navigation
2. **Prepared for growth** — Directory exists when you need to add files
3. **Git requirement** — Git doesn't track empty directories without a file

**Don't delete empty directories** — they're intentional.
