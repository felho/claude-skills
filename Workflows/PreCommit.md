# PreCommit Workflow

**Trigger:** Before running `git commit`.

## Prerequisites

- You have changes ready to commit
- Tests have been run (if applicable)

## Step 1: Verify Commit Coherence

Before every commit, review staged/unstaged changes and ask: "Is this ONE logical change?"

**Think deeply — apply this test:**
> "If I were explaining these changes to someone, would I describe them as ONE thing?
> Or would I naturally say 'and also...' or 'separately...'?"

**Common trap:** Changes from one conversation/session feel like "one thing" but are often multiple logical changes. Ask yourself:
- Did I discover something new mid-task that I then documented? → Separate commit
- Did I fix/update something unrelated while working? → Separate commit
- Are these changes about different concerns (e.g., schedule vs. permissions vs. documentation)? → Separate commits

If multiple unrelated changes are present, use `git add -p` to separate them and commit in multiple steps.

**This applies EVEN within a single file.** If one file contains multiple unrelated changes, split them using `git add -p`.

**Example:**
```bash
# CLAUDE.md has two unrelated changes:
# 1. Added gdrive to monorepo structure
# 2. Added design doc guidelines

# Use git add -p to stage only the first hunk
echo -e "y\nn\n" | git add -p CLAUDE.md
git commit -m "docs(CLAUDE.md): add gdrive CLI to monorepo structure"

# Then stage and commit the rest
git add CLAUDE.md
git commit -m "docs(CLAUDE.md): add design doc guidelines"
```

## Step 2: Watch for WIP Files

When fixing infrastructure issues or other problems mid-task, be careful not to commit work-in-progress files together with the fix. Stage and commit only the files that belong to each logical change.

## Step 3: Pre-Commit Checklist

⚠️ **STOP before committing and verify:**

- [ ] **One logical change:** Review all modified files — are these ALL part of ONE logical change? Apply the "and also" test: if explaining these changes requires saying "and also" or "separately," split into multiple commits.
- [ ] **Tests pass**
- [ ] **Typecheck passes** (`tsc --noEmit` or project-specific command; skip if no `tsconfig.json`)
- [ ] **Plan updated** (checkboxes, counter, current step)
- [ ] **Docs sync:** Which docs reference the changed behavior? Check ALL triggers:
  - New command → app's `README.md`
  - New/changed flag or option → command docs, workflow docs
  - Changed skip/filter/classification logic → workflow docs (e.g., `docs/*/README.md`, `docs/*-workflow.md`)
  - Changed type/interface → design doc
  - Answer: list affected doc files, or write "None found after checking: [files checked]"
- [ ] **Setup sync:** If ran `chmod`, `mkdir -p`, `npm link` → update `scripts/setup.sh`
- [ ] **Docs review (MANDATORY for code changes):** Review ALL `docs/*.md` files read during this session. Update if stale.
- [ ] **Reflection:** Check ALL triggers below

## Step 4: Reflection Triggers

**ALWAYS write out and check each one:**

- [ ] Manual test found something unit tests missed
- [ ] Had to retry or refactor something
- [ ] Discovered a gap that should have been caught earlier
- [ ] Any "huh, that's odd" moment

### If ANY Trigger is Checked

1. **STOP before committing** — do not proceed with git commit yet
2. Write what you learned in your response
3. **Propose a specific action to the user:**
   - If reusable process improvement → "Should I add this to CLAUDE.md?"
   - If coverage gap → "Should I add tests for this?"
   - If documentation gap → "Should I update [specific doc]?"
4. Wait for user response before committing
5. If user approves: implement the improvement, then commit (may need separate commits)
6. If user declines: note the insight in the commit message as a one-off learning

### If NO Triggers Apply

Commit and move on.

> **WHY THIS MATTERS:** Learnings discovered during implementation are valuable.
> If we don't capture them immediately, they're lost. The user may not realize
> a learning occurred unless we explicitly surface it and propose action.

## Plan Management

### Update Implementation Plan

After completing each step, ALWAYS update the plan:
1. Mark completed checkboxes (`- [x]`)
2. Update the "Steps Done" counter
3. Update "Current Step" to point to the next step
4. Include the plan update in the same commit as the step implementation

### Temporary Plans for Complex Steps

When a main plan step requires detailed implementation:

**Workflow:**
1. Create temporary plan: `plans/<feature-name>.md` with detailed steps
2. Work through all steps in the temporary plan
3. When all implementation is done:
   - **Commit 1 (implementation):** All code changes + temporary plan marked as "✓ Complete"
4. **Commit 2 (cleanup):**
   - Delete the temporary plan: `git rm plans/<feature-name>.md`
   - Update main plan: mark step as resolved

### New Command = README Update

When adding a new CLI command or subcommand, ALWAYS check the app's `README.md` before committing. New commands must be documented.

## Decision Gate

Before running `git commit`:

```
"Pre-commit checklist for [commit message]:
- [ ] One logical change: [yes/no]
- [ ] Tests pass: [yes/no/N/A]
- [ ] Typecheck passes: [yes/no/N/A — no tsconfig.json]
- [ ] Plan updated: [yes/no/N/A]
- [ ] Docs sync: [list affected docs OR "None — checked: {files checked}"]
- [ ] Setup sync: [yes/no/N/A]
- [ ] Docs review: [list reviewed docs OR "No docs read this session"]
- [ ] Reflection triggers: [none checked / X checked → action taken]

Ready to commit."
```
