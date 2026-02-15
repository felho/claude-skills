---
name: ManageImpStep
description: Drive multi-step plan implementation with focused context packets. USE WHEN step prepare OR step check OR step execute OR step validate OR step fix OR step done OR implementation workflow OR plan packet.
allowed-tools: Read
---

# ManageImpStep

Structured workflow for implementing multi-step plans. Creates focused "step packets" with all context needed for implementation, tracks status directly in plan files.

## Core Concepts

- **Step Packet:** Standalone implementation prompt with everything needed (from plan + design doc)
- **Stable IDs:** Steps identified by `{phase-id}/{step-id}` via HTML comments in plan
- **Status Tracking:** `<!-- id: step-id status: in-progress|done -->` in plan file

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **Prepare** | `/ManageImpStep prepare <plan> <design-doc> [step-id] [--auto-check]` | `Workflows/Prepare.md` |
| **Check** | `/ManageImpStep check <packet> [--orchestrate] [--dc-pass] [--loop] [--double-check] [--max-cycles N]` | `Workflows/Check.md` |
| **Execute** | `/ManageImpStep execute <packet>` | `Workflows/Execute.md` |
| **Validate** | `/ManageImpStep validate <packet>` | `Workflows/Validate.md` |
| **Fix** | `/ManageImpStep fix <packet> [extra-feedback]` | `Workflows/Fix.md` |
| **Done** | `/ManageImpStep done <packet> [--no-commit]` | `Workflows/Done.md` |

## Workflow Flow

```
PREPARE --auto-check → CHECK --orchestrate (background Task agent)
                         │
                         ├── single-pass iterations (background Tasks)
                         └── dc-pass evaluation (background Task)
                                       ↓
PREPARE → CHECK (optional) → EXECUTE → VALIDATE ──→ DONE
   │                                      ↑    ↓
   └── creates packet                     └── FIX
```

## Status Values

| State | HTML Comment |
|-------|--------------|
| Todo | `<!-- id: step-id -->` (no status) |
| Prepared | `<!-- id: step-id status: prepared -->` (packet ready) |
| In Progress | `<!-- id: step-id status: in-progress -->` |
| Done | `<!-- id: step-id status: done -->` |

## Packet Location

Given plan `plans/foo/bar.md` and step `phase-id/step-id`:
→ Packet: `plans/foo/bar-steps/phase-id/step-id.md`

## Examples

**Example 1: Start implementing next step**
```
User: "/ManageImpStep prepare plans/personal-finance/personal-finance.md docs/personal-finance/README.md"
→ Invokes Prepare workflow
→ Finds first todo step, creates packet, marks in-progress
→ Output: "Prepared step cli-foundation/config-loading. Packet: plans/.../config-loading.md"
```

**Example 2: Prepare specific step**
```
User: "/ManageImpStep prepare plans/myplan.md docs/myplan/README.md import-session/excel-parser"
→ Invokes Prepare workflow for specific step
→ Creates packet for excel-parser step
```

**Example 3: Full implementation cycle**
```
User: "/ManageImpStep prepare ..." → creates packet
User: "/ManageImpStep check <packet>" → verifies packet completeness (single-pass)
User: "/ManageImpStep execute <packet>" → implements the step
User: "/ManageImpStep validate <packet>" → runs tests, checks criteria
User: "/ManageImpStep done <packet>" → marks done, commits (default)
```

**Example 4: Fix failed validation**
```
User: "/ManageImpStep validate <packet>" → FAIL, writes findings file
User: "/ManageImpStep fix <packet>" → reads findings, applies targeted TDD fixes
User: "/ManageImpStep validate <packet>" → PASS, deletes findings file
User: "/ManageImpStep done <packet>" → marks done, commits
```

**Example 5: Single step with auto-check (background)**
```
User: "/ManageImpStep prepare plans/myplan.md docs/myplan/README.md --auto-check"
→ Normal single-step prepare
→ After packet creation, spawns background Task agent running Check --orchestrate
→ Reports immediately with background output file path
→ Check runs asynchronously: single-pass iterations + dc-pass evaluation
```

**Example 6: Check with orchestrator (iterative + double-check)**
```
User: "/ManageImpStep check plans/myplan-steps/phase/step.md --orchestrate"
→ Phase A: spawns single-pass iterations as background Tasks until convergence or 10 iterations
→ Phase B: spawns dc-pass evaluation as background Task (7 dimensions)
→ If any dimension WEAK/MISSING: fix, restart Phase A (max 3 cycles)
→ Updates frontmatter with check-confidence: double-checked
```

**Example 7: Check single-pass (manual, one iteration)**
```
User: "/ManageImpStep check plans/myplan-steps/phase/step.md"
→ One pass: read source docs → analyze gaps → fix → report
→ No looping, no double-check — fast manual review
```

**Example 7B: Legacy flags (backwards compat)**
```
User: "/ManageImpStep check plans/myplan-steps/phase/step.md --loop --double-check"
→ Maps to --orchestrate internally
→ Same behavior as Example 6
```

**Example 8: Resuming a prepared step**
```
User: "/ManageImpStep prepare plans/myplan.md docs/myplan/README.md"
→ First non-done step has status: prepared
→ Sets it to in-progress, skips packet generation
→ Output: "Using pre-generated packet for step-id. Packet: plans/.../step-id.md"
```

**Example 9: Parallel preparation (multiple Claude instances)**
```
# Run in separate Claude Code instances:
Instance 1: "/ManageImpStep prepare plans/myplan.md docs/design.md phase/step-a --auto-check"
Instance 2: "/ManageImpStep prepare plans/myplan.md docs/design.md phase/step-b --auto-check"
→ Each instance has full Skill tool → hooks fire correctly
→ Use explicit step-id to avoid conflicts
```

## References

- [Design Document](references/Design.md) — Full design rationale and edge cases
