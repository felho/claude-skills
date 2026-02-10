# Commit Rules

Detailed rules for creating clean, logical commits.

## One Commit = One Logical Change

Each commit should represent exactly ONE logical change. This applies even when multiple changes are in the same file.

**Rules:**
- If during implementation something unrelated comes up (workflow improvement, refactor, documentation fix), commit it SEPARATELY
- The step implementation + plan update = one commit. Anything else = separate commit(s)
- NEVER bundle unrelated changes together, even if they touch the same files

**When you have multiple logical changes ready to commit:**
1. STOP before running `git add -A` or `git commit`
2. List out the logical changes (e.g., "feature X", "process improvement Y", "bugfix Z")
3. If there are multiple, commit them one by one
4. Use `git add -p` to stage partial file changes when a single file contains multiple logical changes

**Example of what went wrong:**
```
❌ BAD: One commit with:
   - API key config feature (7 files)
   - TodoWrite+TDD rule (CLAUDE.md)
   - git rm rule (CLAUDE.md)

✅ GOOD: Three separate commits:
   1. feat: API key config feature
   2. docs: TodoWrite+TDD rule
   3. docs: git rm rule
```

## Commit Before Switching Context

Before starting a new unrelated change (even if user requests it), check for uncommitted work. If there are uncommitted changes, commit them first before moving to the new task.

## Design Docs and Plans

**Commit immediately after creation:** When creating or updating a design document (`docs/*.md`) or implementation plan (`plans/*.md`):
1. Write the document
2. **Immediately commit it** — do NOT ask user questions first
3. Only THEN ask if ready to start implementation

The plan creation and commit are ONE atomic action. Never leave a plan uncommitted while asking the user what to do next.

**Exception:** Progress tracking updates (marking checkboxes, updating step counters) should be committed **together with** the implementation code they describe — these are one logical change.

## One Design Doc = One Concern

Each design document should focus on a single component or concern. When designing multiple parts of the system simultaneously:
- Create separate documents for each component (e.g., `docs/gmail/send-command.md`, `docs/gdrive/README.md`)
- Keep workflow/orchestration docs separate from component docs
- Each CLI app should have its own `docs/<app>/` directory
- Cross-references between docs are fine, but avoid mixing unrelated designs in one file

**Example of what went wrong:**
```
❌ BAD: Single "upload-workflow-design.md" containing:
   - Gmail send command design
   - New gdrive CLI design
   - WBC orchestration workflow

✅ GOOD: Three separate documents:
   - docs/gmail/send-command.md
   - docs/gdrive/README.md
   - docs/wbc-monthly-close/upload-workflow.md
```

## Check Standards

Before finalizing any design document, check `docs/shared/` for relevant standards and patterns. If a standard exists, ensure your design is compliant.

**Standards to check:**
- `docs/shared/cli-patterns.md` — Exit codes, config paths, global flags
- `docs/shared/testing-strategy.md` — Test structure and conventions
