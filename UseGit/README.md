# UseGit

A Claude Code skill that enforces clean git practices and commit hygiene. It ensures every commit represents exactly one logical change, prevents accidental bundling of unrelated modifications, and provides a comprehensive pre-commit checklist with built-in reflection triggers.

## Why This Skill Exists

Without discipline, git history quickly becomes a mess of "fixed stuff" and "updated things" commits that bundle unrelated changes together. This makes code review harder, `git bisect` useless, and rollbacks risky. UseGit solves this by giving Claude Code a structured process to follow before every commit — checking for coherence, running validations, and even pausing to reflect on what was learned during implementation.

## How It Works

UseGit activates automatically whenever you're committing code or performing git operations. It provides:

1. **A pre-commit workflow** that runs before every `git commit`
2. **Git command rules** that prevent common mistakes (wrong paths, missing renames, etc.)
3. **Commit rules** that enforce one-logical-change-per-commit discipline

## Key Concepts

### One Commit = One Logical Change

The core principle. Before every commit, UseGit asks: "If I were explaining these changes to someone, would I describe them as ONE thing? Or would I naturally say 'and also...'?"

If the answer is "and also," the changes must be split into separate commits — even if they're in the same file.

### The Pre-Commit Checklist

Every commit goes through this checklist:

- **One logical change** — Are all modified files part of ONE logical change?
- **Tests pass** — Do all tests still pass?
- **Typecheck passes** — Does `tsc --noEmit` succeed?
- **Plan updated** — Are implementation plan checkboxes and counters current?
- **Docs sync** — Do any docs reference the changed behavior?
- **Setup sync** — If `chmod`/`mkdir`/`npm link` was run, is `setup.sh` updated?
- **Docs review** — Are all docs read during this session still accurate?
- **Reflection triggers** — Did anything unexpected happen?

### Reflection Triggers

After each commit, UseGit checks for learning opportunities:

- Did a manual test find something unit tests missed?
- Did you have to retry or refactor something?
- Was there a gap that should have been caught earlier?
- Any "huh, that's odd" moments?

If any trigger fires, UseGit pauses to surface the learning and propose a concrete action (add to CLAUDE.md, write a test, update docs) before moving on.

### Partial Staging with git-stage-hunks

When a single file contains multiple unrelated changes, UseGit uses a custom `git-stage-hunks` tool to selectively stage individual hunks:

```bash
# List hunks in a file
~/.claude/tools/git-stage-hunks list CLAUDE.md

# Stage only hunk 1
~/.claude/tools/git-stage-hunks stage CLAUDE.md 1

# Commit the first logical change
git commit -m "docs: add design doc guidelines"

# Stage and commit the rest
git add CLAUDE.md
git commit -m "docs: add gdrive to monorepo structure"
```

## Directory Structure

```
UseGit/
├── SKILL.md                        # Main skill definition and routing
├── README.md                       # This file
├── Workflows/
│   └── PreCommit.md                # Pre-commit checklist and workflow
├── references/
│   ├── CommitRules.md              # One logical change, design docs, standards
│   └── GitCommands.md              # git mv, git rm, repo root, staging rules
└── Tools/
    └── .gitkeep
```

## Usage Examples

### Example 1: Normal commit after completing a feature

```
User: "Commit this change"
→ UseGit runs git status
→ Reviews all changes for coherence
→ Runs pre-commit checklist
→ Creates commit with descriptive message
→ STOPS and waits for user before starting next task
```

### Example 2: Multiple unrelated changes detected

```
User: "Commit"
→ UseGit detects: API feature (7 files) + CLAUDE.md rule update + unrelated bugfix
→ Uses git-stage-hunks to separate changes
→ Creates 3 separate commits, each with one logical change
```

### Example 3: Reflection trigger fires

```
User: "Commit"
→ Pre-commit checklist passes
→ Reflection: "Had to retry the API call test 3 times due to race condition"
→ UseGit pauses: "Should I add a test helper for async retries?"
→ User approves → implements helper → separate commit
```

## Git Command Quick Reference

| Rule | Do This | Not This |
|------|---------|----------|
| Rename files | `git mv old new` | `mv old new && git add .` |
| Delete files | `git rm file` | `rm file` |
| Run git commands | From repo root | From subdirectories |
| Before staging | `git status` first | Assume filenames |
| Stage + commit | Separate commands | Chained with `&&` |

## Installation

This skill is part of the `~/.claude/skills/` directory and activates automatically when Claude Code detects git operations. No additional setup is required.

## When Does It Activate?

UseGit triggers on:
- Committing code
- Creating commits
- Any git operations

The skill description in SKILL.md uses `USE WHEN committing code OR creating commits OR git operations` to match these triggers.
