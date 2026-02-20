# Git Command Rules

Detailed rules for git commands in this project.

## File Operations

### Always use `git mv` for file renames

When renaming or moving files, ALWAYS use `git mv old_path new_path` instead of `mv`.

**Why:** This ensures Git properly tracks the file history. While Git can sometimes detect renames after staging (if content similarity is >50%), using `git mv` is explicit and reliable.

```bash
# ✅ CORRECT
git mv old-name.ts new-name.ts

# ❌ WRONG
mv old-name.ts new-name.ts
git add .
```

### Always use `git rm` for file deletions

When deleting tracked files, ALWAYS use `git rm path` instead of `rm`.

**Why:** This stages the deletion in one step. Using `rm` alone leaves the deletion unstaged and requires a separate `git add` to track it.

```bash
# ✅ CORRECT
git rm obsolete-file.ts

# ❌ WRONG
rm obsolete-file.ts
git add obsolete-file.ts
```

## Command Execution

### Always run git commands from repo root

When running `git add`, `git commit`, or other git commands, ALWAYS `cd` to the repo root first.

**Why:** Running git commands from subdirectories with repo-root-relative paths causes "pathspec did not match any files" errors (exit code 128).

```bash
# ✅ CORRECT: Run from repo root
cd /Users/felho/dev/wbc-monthly-close && git add apps/wbc-monthly-close/src/file.ts

# ❌ WRONG: Running from app directory with repo-relative path
cd apps/wbc-monthly-close && git add apps/wbc-monthly-close/src/file.ts  # exit code 128
```

### Always run `git status` before staging

NEVER assume filenames — always run `git status` first and use the exact filenames shown.

**Why:** This prevents "pathspec did not match" errors from typos or wrong assumptions (e.g., `bun.lockb` vs `bun.lock`).

```bash
# ✅ CORRECT: Check status first, then use exact filenames
cd /Users/felho/dev/wbc-monthly-close && git status
# See: modified: bun.lock (not bun.lockb!)
git add bun.lock

# ❌ WRONG: Assuming filename without checking
git add bun.lockb  # exit code 128 - file doesn't exist
```

### Separate `git add` and `git commit`

ALWAYS run `git add` and `git commit` as separate commands (not chained with `&&`).

**Why:** This makes it easier to review what's being staged vs committed in the conversation. You can proceed directly from `git add` to `git commit` without waiting for user review.

```bash
# ✅ CORRECT: Separate commands
git add apps/gdrive/src/index.ts plans/gdrive.md
git commit -m "feat(gdrive): implement feature X"

# ❌ WRONG: Chained commands
git add apps/gdrive/src/index.ts && git commit -m "..."
```

## Partial Staging (Multiple Logical Changes in One File)

### When to use partial staging

When a single file contains changes that belong to **different logical commits**, you MUST stage them separately using `~/.claude/tools/git-stage-hunks`.

**Signs you need partial staging:**
- One file has both a bug fix and a new feature
- Documentation updates mixed with code changes
- A workflow file has both instruction changes and implementation changes
- A plan file has status changes for multiple steps

### Primary method: `git-stage-hunks` tool

The `~/.claude/tools/git-stage-hunks` tool provides non-interactive selective hunk staging. It replaces both `git add -p` (which requires interactive input) and manual patch file creation (which is error-prone).

**Commands:**
- `~/.claude/tools/git-stage-hunks list <file>` — show all hunks with numbers, +/- stats, preview
- `~/.claude/tools/git-stage-hunks stage <file> <hunks>` — stage specific hunks (e.g., `1`, `1,3`, `1-3`)
- `~/.claude/tools/git-stage-hunks diff <file> <hunks>` — dry-run, show the patch without applying

**How it works:** Extracts hunks from `git diff`, assembles a patch with only the selected hunks, and applies it via `git apply --cached`. The remaining hunks stay unstaged.

### Example: Separating two unrelated changes

```bash
# File has: (1) skill reactivation reminder (hunk 1), (2) row calculation docs (hunk 2)

# Step 1: List hunks to see what's where
~/.claude/tools/git-stage-hunks list Workflows/Example.md
# Output shows:
#   Hunk 1: +skill reactivation reminder
#   Hunk 2: +row calculation docs

# Step 2: Stage and commit hunk 1
~/.claude/tools/git-stage-hunks stage Workflows/Example.md 1
git commit -m "docs: add skill reactivation reminder"

# Step 3: Stage and commit remaining changes
git add Workflows/Example.md
git commit -m "docs: add row calculation documentation"
```

### Example: Plan file with multiple step status changes

```bash
# Plan has status changes for two different steps
~/.claude/tools/git-stage-hunks list plans/myplan.md
# Hunk 1: <!-- id: step-a status: in-progress -->
# Hunk 2: <!-- id: step-b status: done -->

# Stage only the step-a change
~/.claude/tools/git-stage-hunks stage plans/myplan.md 1
git add plans/myplan-steps/phase/step-a.md
git commit -m "chore: prepare step phase/step-a"
```

### Fallback: Manual patch file

If `git-stage-hunks` cannot handle a case (e.g., changes within the same hunk that need splitting), fall back to creating a manual patch:

```bash
# 1. Get the full diff
git diff -- path/to/file.ts > /tmp/full.patch
# 2. Manually edit to keep only desired lines
# 3. Apply to staging area
git apply --cached /tmp/edited.patch
```
