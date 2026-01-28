# Step-by-Step Implementation Workflow

> **Status:** Design phase — prototyping Claude Code commands

## Overview

A set of Claude Code slash commands to drive implementation of multi-step plans in a structured, repeatable way.

## Problem Statement

When implementing a large plan (like `plans/personal-finance/personal-finance.md` with 57 steps across 10 phases), we need:

1. **Focused context** — Each step needs only relevant information, not the entire plan
2. **Quality prompts** — Implementation instructions must be clear and complete
3. **Tracking** — Know which step is next, what's done, what's in progress
4. **Resilience** — Handle plan changes during implementation
5. **Manual control** — Human decides when to proceed, not full automation

## Background & Brainstorming

### The Core Workflow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   PREPARE   │ ──► │   CHECK     │ ──► │   EXECUTE   │
│  (packet)   │     │  (quality)  │     │   (impl)    │
└─────────────┘     └─────────────┘     └─────────────┘
                                               │
                                               ▼
                    ┌─────────────┐     ┌─────────────┐
                    │    DONE     │ ◄── │  VALIDATE   │
                    │  (mark it)  │     │  (verify)   │
                    └─────────────┘     └─────────────┘

Note: `/ManageImpStep check` is optional but recommended. Minimum flow: PREPARE → EXECUTE → VALIDATE → DONE
```

### Key Design Decisions

#### 1. Stable Step Identification

**Problem:** Steps identified by number (e.g., "Step 2.3") break when plan is reordered.

**Solution:** Assign stable semantic IDs via HTML comments:

```markdown
## Phase 2: Import & Session Management
<!-- id: import-session -->

### Step 2.3: Excel Parser Library
<!-- id: excel-parser -->
```

**Full identifier:** `{phase-id}/{step-id}` → `import-session/excel-parser`

These IDs never change even if steps are renumbered or reordered.

#### 2. Step Packet (Detailed Step Implementation)

**Problem:** Claude needs focused context, not the entire plan + design doc.

**Solution:** Generate a "step packet" — a standalone implementation prompt containing everything needed:

```markdown
---
step: cli-foundation/config-loading
plan: plans/personal-finance/personal-finance.md
design-doc: docs/personal-finance/README.md
created: 2026-01-27T10:00:00Z
---

# Step: cli-foundation/config-loading

## Overview
[Goal + value statement]

## Step Definition
[Full step text from implementation plan]

## From Design Document
[All relevant sections from design doc]

## From Plan (Supporting Sections)
[Actual content of any supporting sections from the plan — e.g., error message guidelines, type definitions, related steps. Content is physically included, not just referenced, to ensure it's in the LLM context.]

Note: Claude analyzes the step definition for references (e.g., "see section X", "spec 4.1", "as defined in...") and includes the referenced content. This is prompt-dependent — the `/ManageImpStep check` workflow helps catch missing context.

## Dependencies
[Prerequisite steps that must be complete]

## Test Cases
[Specific test scenarios]

## Acceptance Criteria
[What must be true for the step to be considered complete]

## Implementation Notes
[Gotchas, critical notes, exact error messages]
```

**YAML frontmatter purpose:**
- Stores source document paths → `/ManageImpStep check` reads these
- Audit trail → know where packet came from
- Reproducible → can regenerate if needed

**Edge case — plan or design doc moved:** If the paths in frontmatter become stale (files moved/renamed), packet-based commands will error: "Plan file not found: {path}" or "Design document not found: {path}". Fix by updating the paths in the packet's YAML frontmatter.

**Example packet location:** `plans/personal-finance/personal-finance-steps/cli-foundation/config-loading.md`

#### 3. No State Files — Status in Plan

**Considered:** `.claude/step-state.json` to track current step, status, etc.

**Decision:** No separate state files. Track status directly in the plan using HTML comment attributes:

```markdown
### Step 2.3: Excel Parser Library
<!-- id: excel-parser status: in-progress -->
```

**Status values:**

| State | Representation |
|-------|----------------|
| Todo | `<!-- id: excel-parser -->` — no status attribute means todo |
| In Progress | `<!-- id: excel-parser status: in-progress -->` |
| Done | `<!-- id: excel-parser status: done -->` |

The HTML comment (with ID) is always preserved. Only the status attribute changes.

**Expected format:** `<!-- id: {id} -->` or `<!-- id: {id} status: {status} -->` (single spaces, no extra whitespace)

**Source of truth:**
- Plan `status` attribute = **workflow state** (todo/in-progress/done)
- Packet = **implementation specification** (what to build, gathered from plan + design doc)
- Checkboxes within a step are for visibility (see which sub-tasks are done) but don't affect workflow logic
- `/ManageImpStep done` updates both status attribute and checkboxes

**Packet-based operations:**
- `/ManageImpStep check`, `/ManageImpStep execute`, and `/ManageImpStep validate` use the packet as their source of truth
- All three verify the step is `in-progress` in the plan (error if done or todo)
- Only `/ManageImpStep prepare` and `/ManageImpStep done` write the plan's status attribute

**Benefits:**
- Single source of truth (everything in the plan)
- Git tracks status changes (see when work started)
- Human-readable
- Easy to parse programmatically

**Phase definition:** A phase is defined by an H2 heading followed by an `<!-- id: phase-id -->` comment. Phases cannot have status attributes.

**Step definition:** A step is defined by an H3 heading followed by an `<!-- id: step-id -->` comment. Headings without ID comments are not recognized as steps. Only steps can have status attributes.

**Example step in plan:**
```markdown
### Step 1.4: Config Loading Library
<!-- id: config-loading -->
- [ ] Create `src/types/index.ts`
- [ ] Create `src/lib/config.ts`
- [ ] Implement `loadAccounts()`

**Test:** Unit tests for each loader
**Value:** Centralized config access
```

**Note:** Steps without checkboxes are valid — `/ManageImpStep done` will only update the status attribute.

**Step selection logic:** See `/ManageImpStep prepare` behavior for detailed step selection logic.

**Re-running a completed step:**
1. Manually remove `status: done` from the step's HTML comment (step returns to todo status)
2. Run `/ManageImpStep prepare` again to create a new packet

**Edge case — packet exists but step is todo:**
If packet exists but step is todo (e.g., user manually removed `status: in-progress`), `/ManageImpStep prepare` will ask about overwrite and set status to in-progress.

#### 4. Claude Code Commands vs CLI

**Considered:** `pstep` CLI tool with subcommands.

**Decision:** Claude Code slash commands.

**Rationale:**
- We're prototyping prompts, not building infrastructure
- Claude Code commands are faster to iterate
- No build/install cycle
- Can evolve into CLI later if needed

#### 5. Merge "next" into "prepare"

**Considered:** Separate `/ManageImpStep next` (show next) and `/ManageImpStep prepare` (create packet).

**Decision:** Merge into single `/ManageImpStep prepare`.

**Rationale:**
- In practice, seeing next → immediately want to prepare
- Two commands = more friction
- `/ManageImpStep prepare` shows what's next as part of its output

### Folder Structure Convention

When a plan needs step packets, create a sibling folder with phase subfolders:

```
plans/
└── personal-finance/
    ├── personal-finance.md              # The plan with stable IDs
    └── personal-finance-steps/          # Generated packets
        ├── data-setup/                  # Phase subfolder
        │   ├── create-dirs.md
        │   └── export-sheets.md
        ├── cli-foundation/
        │   ├── app-skeleton.md
        │   └── config-loading.md
        └── ...
```

**Naming convention:** Given a plan file `{name}.md`, the steps folder is `{name}-steps/` (filename without extension + `-steps`).

**Path pattern:** `{plan-file-dir}/{name}-steps/{phase-id}/{step-id}.md`

This mirrors the `{phase-id}/{step-id}` identifier format.

---

## Workflows (5 total)

**Note:** `/ManageImpStep` without arguments displays the skill overview (SKILL.md).

### `/ManageImpStep prepare`

**Purpose:** Create detailed step implementation packet from plan + design doc.

**Parameters:**
1. `<plan>` — implementation plan file path (required)
2. `<design-doc>` — design document file path (required)
3. `[full-step-id]` — format: `{phase-id}/{step-id}` (e.g., `import-session/excel-parser`) (optional, defaults to next step)

**Constraint:** Every plan must have an associated design document. This ensures the packet has complete implementation details.

**Usage:**
```
/ManageImpStep prepare plans/personal-finance/personal-finance.md docs/personal-finance/README.md
/ManageImpStep prepare plans/personal-finance/personal-finance.md docs/personal-finance/README.md cli-foundation/config-loading
```

**Behavior:**
1. Read implementation plan
2. Find step to prepare:

   **If step-id provided:**
   - If that step is already in-progress → check if packet exists; if exists, ask about overwrite; if not, create it (status remains in-progress)
   - If that step is done → error
   - If ANOTHER step is in-progress → error
   - Otherwise → prepare that step (set to in-progress, create packet)

   **If no step-id provided:**
   - If there's one in-progress step → check if packet exists:
     - If packet exists → report step info and packet path, suggest next step: "Run `/ManageImpStep execute <packet>` to continue implementation." (no changes, just informational)
     - If packet missing → create packet (status remains in-progress)
   - If multiple in-progress → error
   - Otherwise → first step with todo status (no status attribute)
   - Error cases:
     - Plan file not found → "Plan file not found: {path}"
     - Design doc not found → "Design document not found: {path}"
     - Plan has no valid steps → "No steps found in plan (steps require H3 heading + `<!-- id: ... -->` comment)"
     - Another step is already in-progress → "Step {other-step-id} is already in-progress. Finish or abandon it first (remove `status: in-progress` from the plan)."
     - Multiple `status: in-progress` found → "Multiple steps in-progress. Fix the plan manually."
     - Step has `status: done` → "Step {step-id} is already done. To re-implement, remove `status: done` from the HTML comment, then run `/ManageImpStep prepare`."
     - Step-id not found in plan → "Step not found: {step-id}"
3. Check if packet already exists:
   - If exists → warn and ask user (via AskUserQuestion tool): "Packet already exists. Overwrite?"
   - If user declines → abort
   - If step is already `in-progress` and packet is overwritten, status remains unchanged
4. Extract step definition from plan
5. Gather ALL related information from:
   - Implementation plan (spec references, cross-references, dependencies)
   - Design document (relevant sections, details, edge cases)
6. Mark step as `status: in-progress` in plan
7. Create packet file with YAML frontmatter:
   ```yaml
   ---
   step: cli-foundation/config-loading
   plan: plans/personal-finance/personal-finance.md
   design-doc: docs/personal-finance/README.md
   created: 2026-01-27T10:00:00Z
   ---
   ```
8. Output: step info + packet path

**Artifacts:**
- `{plan}-steps/{phase-id}/{step-id}.md` (detailed step implementation packet)
- Modified plan file (status: in-progress)

---

### `/ManageImpStep check`

**Purpose:** Verify packet completeness against source documents, improve if needed.

**Parameters:**
1. `<packet>` — path to detailed step implementation packet (required)

**Usage:**
```
/ManageImpStep check plans/personal-finance/personal-finance-steps/cli-foundation/config-loading.md
```

**Behavior:**
1. Read packet file
2. Extract source paths from YAML frontmatter (plan, design-doc)
3. Re-read BOTH source documents thoroughly
4. Compare against packet — find EVERYTHING that's missing:
   - Details mentioned in design doc but not in packet
   - Edge cases, error messages, validation rules
   - Cross-references, dependencies
   - Test scenarios
   - Acceptance criteria completeness
5. Update the packet with missing information
6. Output: what was added/changed

**Key prompt:**
> "Think very hard. Read the design document and implementation plan carefully. Does this packet have ALL information needed for successful implementation? Find every missing detail. Add ONLY information that is NOT already in the packet — do not duplicate existing content."

**Error handling:**
- Invalid/missing YAML frontmatter → "Invalid packet: missing required frontmatter fields (step, plan, design-doc)"
- Step is not in-progress (no status attribute) → "Step {step-id} is not in-progress (status may have been manually removed). Run `/ManageImpStep prepare` to resume."
- Step has `status: done` in plan → "Step {step-id} is already done. Remove `status: done` from the HTML comment, then run `/ManageImpStep prepare`."

**Artifact:** Modified packet (can run multiple times, idempotent — won't duplicate info)

---

### `/ManageImpStep execute`

**Purpose:** Run implementation based on packet.

**Usage:**
```
/ManageImpStep execute <packet>                   # Execute the packet
```

**Behavior:**
1. Read packet file, validate YAML frontmatter
2. Check plan status — step must be `in-progress`
3. Follow implementation instructions in packet
4. Write code, create tests, etc.

**Key prompt:**
> "Implement this step exactly as specified in the packet. Follow TDD: write tests first, then implementation. Do not add features beyond what is specified."

**Error handling:**
- Packet file not found → "Packet not found: {path}"
- Invalid/missing YAML frontmatter → "Invalid packet: missing required frontmatter fields"
- Step is not in-progress (no status attribute) → "Step {step-id} is not in-progress (status may have been manually removed). Run `/ManageImpStep prepare` to resume."
- Step has `status: done` in plan → "Step {step-id} is already done. Remove `status: done` from the HTML comment, then run `/ManageImpStep prepare`."

**Note:** Running `/ManageImpStep execute` multiple times is allowed — status remains in-progress.

**Artifact:** Code changes

---

### `/ManageImpStep validate`

**Purpose:** Verify implementation meets acceptance criteria.

**Usage:**
```
/ManageImpStep validate <packet>
```

**Behavior:**
1. Read packet file, extract acceptance criteria
2. Run checks:
   - `tsc --noEmit` (type check)
   - `bun run test` (tests pass)
   - File existence checks
   - Custom criteria from step
3. Output: pass/fail with details

**Key prompt:**
> "Verify the implementation meets ALL acceptance criteria from the packet. Run all specified checks. Report clearly what passed and what failed."

**If validation fails:**
1. Fix the issues manually or re-run `/ManageImpStep execute`
2. Re-run `/ManageImpStep validate` until it passes
3. Do not run `/ManageImpStep done` until validation passes

**Error handling:**
- Invalid/missing packet → "Invalid packet: file not found or missing required frontmatter fields"
- Step is not in-progress (no status attribute) → "Step {step-id} is not in-progress (status may have been manually removed). Run `/ManageImpStep prepare` to resume."
- Step has `status: done` in plan → "Step {step-id} is already done. Remove `status: done` from the HTML comment, then run `/ManageImpStep prepare`."

**Artifact:** None (just reports)

---

### `/ManageImpStep done`

**Purpose:** Mark step complete in plan file.

**Usage:**
```
/ManageImpStep done <packet>                      # Mark step done
/ManageImpStep done <packet> --commit             # Also create git commit
```

**Behavior:**
1. Read packet, get step ID from frontmatter
2. Find step in plan file
3. Check all checkboxes for that step (already checked boxes are left unchanged)
4. Change `status: in-progress` to `status: done`
5. If `--commit`: create git commit

**Key prompt:**
> "Mark this step as complete. Update the plan file: check all task checkboxes and set status to done."

**Important:**
- User's responsibility to run `/ManageImpStep validate` before `/ManageImpStep done`. The `--commit` flag does not re-run validation.
- When `--commit` is used, the workflow follows UseGit skill practices (conventional commit message, pre-commit checklist verification).

**Error handling:**
- Invalid/missing packet → "Invalid packet: file not found or missing required frontmatter fields"
- Step is not in-progress (no status attribute) → "Step {step-id} is not in-progress. Run `/ManageImpStep prepare` first."
- Step has `status: done` → "Step {step-id} is already done. To re-implement, remove `status: done` from the HTML comment, then run `/ManageImpStep prepare`."

**Artifact:** Modified plan file (+ git commit if requested)

---

## Resolved Questions

1. **Packet format:** YAML frontmatter (metadata) + Markdown (content)
2. **Context gathering:** Extract from both implementation plan AND design document
3. **Multiple plans:** Plan path is a mandatory parameter
4. **Partial steps:** Not needed — step is either in-progress or done

## Workflow Scope Guardrails

Each workflow has explicit scope boundaries to prevent common mistakes where Claude does work outside the workflow's intent.

### The Problem

Without clear guardrails, Claude tends to:
- **Prepare:** Start implementing instead of just creating the packet documentation
- **Check:** Fix implementation issues instead of just improving the packet
- **Execute:** Add extra features, refactor adjacent code, or fix unrelated issues
- **Done:** Do "one more fix" before marking complete

### The Solution

Each workflow now includes:

1. **Scope warning** (blockquote at top) — Clear "Do NOT" list specific to that workflow
2. **allowed-tools note** — Clarifies what Write/Edit tools are for
3. **STOP check** (where applicable) — Self-verification checklist before reporting

### Per-Workflow Guardrails

| Workflow | Scope | Can Modify |
|----------|-------|------------|
| **Prepare** | Create packet documentation only | Only the packet `.md` file + plan status |
| **Check** | Enhance packet documentation only | Only the packet `.md` file |
| **Execute** | Implement exactly what packet specifies | Implementation files (code, config, tests) |
| **Validate** | Report pass/fail status only | Nothing (read-only, no Write/Edit tools) |
| **Done** | Mark completion only | Only the plan `.md` file |

### Key Insight

**Validate** is naturally protected — it has no Write/Edit tools, so it physically cannot modify files. The other workflows needed explicit guardrails because they have modification capabilities that could be misused.

### STOP Checks

Two workflows have explicit STOP checks before reporting:

- **Prepare** (Step 10): Verify only packet was created, no implementation work done
- **Execute** (Step 10): Verify stayed in scope, no extra features or out-of-scope fixes

These act as a final self-verification to catch scope violations before completing the workflow.

## Future Enhancements

1. **Plan change detection:** Include hash of step definition in packet. Before execute, compare current definition hash. Warn if plan changed since packet creation. (For now: engineer's responsibility to be aware of plan changes.)

2. **Abandon workflow:** If `/ManageImpStep prepare` runs but step is never completed, it remains `in-progress` indefinitely. Workaround: manually edit the plan to remove `status: in-progress` from the step's HTML comment. Could add `/ManageImpStep abandon` command in the future.

---

## Skill Structure

The step workflow is implemented as a global Claude Code skill with workflows:

```
~/.claude/skills/ManageImpStep/
├── SKILL.md              # Skill definition + overview
└── workflows/
    ├── prepare.md        → /ManageImpStep prepare
    ├── check.md          → /ManageImpStep check
    ├── execute.md        → /ManageImpStep execute
    ├── validate.md       → /ManageImpStep validate
    └── done.md           → /ManageImpStep done
```

**Why this structure:**
- Official Claude Code pattern (skill + workflows)
- `/ManageImpStep` without arguments shows the SKILL.md overview (available workflows and usage)
- `/ManageImpStep <workflow>` runs specific workflow
- Each workflow in separate file — clean separation
- Global availability via `~/.claude/skills/`

## Next Steps

1. Create `~/.claude/skills/ManageImpStep/SKILL.md` with overview
2. Create workflow prompts using `/WritePrompt` skill
   - **Important:** Use the WritePrompt skill when creating each workflow prompt — it has guidance on prompt structure for agentic use cases
3. Test with `personal-finance` plan
4. Iterate on prompts based on what works

---

## References

- Example plan with stable IDs: `plans/personal-finance/personal-finance.md`
- Step packets folder: `plans/personal-finance/personal-finance-steps/{phase-id}/{step-id}.md`
