---
description: Analyze a PRD for complexity and restructure into proper mediums
argument-hint: <prd-path>
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Task
---

# Structure — Analyze, Plan, and Restructure a PRD

Walk through a PRD's content area by area, make medium decisions (code vs prose vs state machine vs config), produce a migration plan, and execute the restructuring step by step.

> This workflow adapts to PRD size: proactive for growing PRDs (~500 lines), retroactive untangling for mature PRDs (~3000+ lines).

## Variables

PRD_PATH: $1
USUAL_SUSPECTS_REF: references/UsualSuspects.md (relative to CraftPRD skill directory)
MEDIUM_GUIDE_REF: references/MediumGuide.md (relative to CraftPRD skill directory)
COMPLEXITY_SIGNALS_REF: references/ComplexitySignals.md (relative to CraftPRD skill directory)
STRUCTURED_SPEC_TEMPLATE: references/Templates/StructuredSpec.md (relative to CraftPRD skill directory)

## Instructions

- If `PRD_PATH` is empty → STOP with "Usage: provide a path to the PRD file."
- **Path resolution (MUST do first):** `PRD_PATH` may be relative. Resolve it to an absolute path before ANY tool call:
  1. If `PRD_PATH` is already absolute (starts with `/`) → use as-is.
  2. If relative → resolve against the **current working directory** (the directory Claude Code was launched from). Use `Glob` with the relative pattern to find the actual file. If Glob finds exactly one match, use that absolute path. If zero matches, try common prefixes (`~/`, `./`) before reporting "not found".
  3. **Update `PRD_PATH`** to the resolved absolute path for all subsequent steps.
- This is the most complex workflow. It has 4 phases: Analyze, Discuss, Plan, Execute.
- **Do NOT skip the Discuss phase.** Every area needs an explicit user decision before migration.
- Write all output files in **English**.
- The design doc (prose) survives restructuring — it shrinks to decisions, rationale, and flows. Technical contracts move to code.
- Execute the migration **step by step with commits after each step.** Do not batch everything.

## Workflow

### Phase 1: ANALYZE

#### 1.1 Read Source Documents

- Read the PRD at `PRD_PATH`.
  - If not found → STOP with "PRD not found: {PRD_PATH}"
- Read `USUAL_SUSPECTS_REF`, `MEDIUM_GUIDE_REF`, and `COMPLEXITY_SIGNALS_REF`.

#### 1.2 Generate Complexity Report

Scan the PRD and produce a quantitative report:

```
Complexity Analysis: <PRD name>

Quantitative:
- Total lines: <N>
- TypeScript interfaces in markdown: <N>
- SQL/schema blocks: <N>
- Cross-references ("see section"): <N>
- Repeated enums (same values in 2+ places): <N>
- ASCII diagrams: <N>

Qualitative signals detected:
- <signal description, if present>
```

#### 1.3 Map Areas Present in PRD

For each of the 12 "usual suspects" from `USUAL_SUSPECTS_REF`, check if the PRD contains content for that area. Produce a checklist:

```
Areas detected in PRD:
[x] Core Domain Model — ~N lines, M interfaces
[x] State Machine — ~N lines, ASCII diagram present
[ ] API Contract — not found
[x] Communication Protocol — ~N lines
...
```

Present this to the user before proceeding.

### Phase 2: DISCUSS (area by area)

For each detected area, walk through three questions with the user:

<area-discussion-loop>

**Question 1: "Is this area present and significant?"**
Confirm the detection. Show the relevant PRD lines/sections. The user may disagree with the assessment.

**Question 2: "Core or commodity?"**
- Core = unique to this product, must be custom-built.
- Commodity = existing tools/frameworks can handle it.

If commodity → discuss framework options (from `USUAL_SUSPECTS_REF`). The user decides which framework/tool.

**Question 3: "What medium?"**
Using `MEDIUM_GUIDE_REF`, recommend the target medium. Options:
- **Stays in prose** (design doc) — decisions, rationale, flows
- **Moves to code** (TypeScript) — types, schemas, contracts, invariants
- **Moves to state machine** (XState/table) — transitions, guards, side effects
- **Moves to config** (YAML) — tuneable parameters
- **Eliminated by framework** — replaced by framework choice, PRD section shrinks to "we use X"

Record the decision for each area.

</area-discussion-loop>

After all areas are discussed, present the decisions summary to the user for final confirmation.

### Phase 3: PLAN

#### 3.1 Generate Migration Plan

Based on the decisions, create a concrete migration plan mapping PRD sections to target files:

```markdown
# Migration Plan: <PRD name>

## Decisions Summary

| Area | Decision | Target |
|------|----------|--------|
| Core Domain Model | Move to code | packages/domain/types.ts |
| State Machine | Move to state machine | packages/domain/state-machine.ts |
| Communication Protocol | Use Socket.IO | PRD section → "We use Socket.IO" |
| ... | ... | ... |

## Migration Steps (in order)

### Step 1: Create target directory structure
- Create packages/domain/, packages/protocol/, etc.

### Step 2: Extract core domain types
- Source: PRD lines N-M (section "Data Model")
- Target: packages/domain/types.ts
- PRD action: Replace section with reference

### Step 3: Extract state machine
- Source: PRD lines N-M (section "Status State Machine")
- Target: packages/domain/state-machine.ts
- PRD action: Replace diagram and prose with reference

...

## Expected Result
- Design doc: ~N lines (from ~M lines)
- New files: <list>
- Frameworks adopted: <list>
```

#### 3.2 Read Structured Spec Template

If `STRUCTURED_SPEC_TEMPLATE` exists, use it as a guide for the target directory structure.

#### 3.3 Present Plan to User

Show the migration plan. Ask for approval before executing. The user may want to reorder steps or skip some.

### Phase 4: EXECUTE (step by step)

<migration-execute-loop>

For each step in the approved migration plan:

1. **Announce** the step: "Step N: <description>"
2. **Extract** content from PRD (read relevant sections)
3. **Create** target file with the extracted content, transformed into the proper medium
4. **Update** PRD: replace extracted section with a reference (e.g., "See `packages/domain/state-machine.ts` for full transition rules.")
5. **Verify** the change doesn't break cross-references in the PRD
6. **STOP and wait** for the user to review and commit before proceeding to the next step

</migration-execute-loop>

### 5. Final Report

After all steps are complete, generate a summary.

## Report

```
Structure complete: <PRD name>

Before: <N> lines
After: <M> lines (<reduction>% reduction)

Files created:
- <path>: <description>
- <path>: <description>

Frameworks adopted:
- <framework>: replaces <what>

Design doc now contains:
- <section list — should be mostly decisions, rationale, flows>

Next: Run the Review workflow to verify consistency across the restructured spec.
```
