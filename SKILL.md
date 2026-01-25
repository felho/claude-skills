---
name: UseGit
description: Git practices for this project. USE WHEN committing code OR creating commits OR git operations. Enforces clean git history.
---

# UseGit

Enforces clean git practices and commit hygiene for this project.

## Core Principles

> **One commit = one logical change.**

> **Commit after each step, then STOP and wait for user.**

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **PreCommit** | Before running `git commit` | `Workflows/PreCommit.md` |

## Quick Reference

### Git Command Rules

| Rule | Command |
|------|---------|
| File renames | `git mv` (not `mv`) |
| File deletions | `git rm` (not `rm`) |
| Run commands from | Repo root (use `git rev-parse --show-toplevel`) |
| Before staging | Always run `git status` first |
| Staging vs committing | Separate `git add` and `git commit` commands |

### Commit Rules

1. **One logical change per commit** - Don't bundle unrelated changes
2. **Commit after each step** - Before starting next task
3. **Stop after committing** - Wait for user to request next step
4. **Include plan updates** - Step implementation + plan update = one commit

## Examples

**Example 1: After completing a feature step**
```
[Implementation done, tests pass]
→ git status
→ git add [files]
→ git commit -m "feat: ..."
→ STOP and wait for user
```

**Example 2: Renaming a file**
```
→ git mv old-name.ts new-name.ts
→ git commit -m "refactor: rename ..."
```

**Example 3: Multiple logical changes found**
```
[About to commit, notice unrelated changes]
→ STOP
→ Use git add -p to stage only related changes
→ Commit each logical change separately
```

## References

- [GitCommands.md](references/GitCommands.md) - Detailed git command rules with examples
- [CommitRules.md](references/CommitRules.md) - One logical change, design docs, standards
