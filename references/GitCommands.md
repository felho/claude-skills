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

When a single file contains changes that belong to **different logical commits**, you MUST stage them separately using patch mode or manual patches.

**Signs you need partial staging:**
- One file has both a bug fix and a new feature
- Documentation updates mixed with code changes
- A workflow file has both instruction changes and implementation changes

### Method 1: Create a patch file for specific lines

When you need to stage only specific lines from a file:

```bash
# 1. Create a patch file with only the lines you want
cat > /tmp/partial.patch << 'PATCH'
diff --git a/path/to/file.md b/path/to/file.md
--- a/path/to/file.md
+++ b/path/to/file.md
@@ -10,6 +10,8 @@ Some context line
 Another context line

+The specific line(s) you want to add
+
 More context
PATCH

# 2. Apply patch to staging area only
git apply --cached /tmp/partial.patch

# 3. Verify what's staged
git diff --cached path/to/file.md

# 4. Commit the partial change
git commit -m "docs: add specific change"

# 5. Stage and commit remaining changes
git add path/to/file.md
git commit -m "feat: add other changes"
```

### Method 2: Interactive staging with `git add -p`

For simpler cases where hunks are already separate:

```bash
git add -p path/to/file.md
# y = stage this hunk
# n = skip this hunk
# s = split hunk into smaller hunks
# e = manually edit the hunk
```

**Note:** `git add -p` works well when changes are in different parts of the file. When changes are interleaved or in the same hunk, use Method 1 (patch file).

### Example: Separating instruction changes from workflow improvements

```bash
# File has: (1) skill reactivation reminder, (2) row calculation docs

# Commit 1: Stage only the skill reactivation line
cat > /tmp/skill-reactivation.patch << 'PATCH'
diff --git a/Workflows/Example.md b/Workflows/Example.md
--- a/Workflows/Example.md
+++ b/Workflows/Example.md
@@ -50,6 +50,8 @@ Some existing content

+> **🛑 SKILL REACTIVATION REQUIRED:** Reminder text here.
+
 More existing content
PATCH

git apply --cached /tmp/skill-reactivation.patch
git commit -m "docs: add skill reactivation reminder"

# Commit 2: Stage remaining changes
git add Workflows/Example.md
git commit -m "docs: add row calculation documentation"
```
