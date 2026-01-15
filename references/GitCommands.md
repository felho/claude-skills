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
