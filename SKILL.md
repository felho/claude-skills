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
| **Prepare** | `/ManageImpStep prepare <plan> <design-doc> [step-id] [--ahead N] [--auto-check]` | `Workflows/Prepare.md` |
| **Check** | `/ManageImpStep check <packet>` | `Workflows/Check.md` |
| **Execute** | `/ManageImpStep execute <packet>` | `Workflows/Execute.md` |
| **Validate** | `/ManageImpStep validate <packet>` | `Workflows/Validate.md` |
| **Fix** | `/ManageImpStep fix <packet> [extra-feedback]` | `Workflows/Fix.md` |
| **Done** | `/ManageImpStep done <packet> [--no-commit]` | `Workflows/Done.md` |

## Workflow Flow

```
PREPARE --ahead --auto-check → (packets created + checked, status: prepared)
                                       ↓ (later)
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
User: "/ManageImpStep check <packet>" → verifies packet completeness
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

**Example 5: Prepare-ahead with auto-check**
```
User: "/ManageImpStep prepare plans/myplan.md docs/myplan/README.md --ahead 3 --auto-check"
→ Invokes Prepare workflow in ahead mode
→ Finds next 3 todo steps, launches parallel agents
→ Each agent: creates packet + runs check loop until clean
→ Sets status: prepared on successful steps
→ Output: "Prepared 3 steps: step-a, step-b, step-c"
```

**Example 6: Prepare-ahead without auto-check**
```
User: "/ManageImpStep prepare plans/myplan.md docs/myplan/README.md --ahead 2"
→ Finds next 2 todo steps, creates packets in parallel
→ No check loop — packets may have gaps
→ Sets status: prepared on successful steps
```

**Example 7: Single step with auto-check**
```
User: "/ManageImpStep prepare plans/myplan.md docs/myplan/README.md --auto-check"
→ Normal single-step prepare
→ After packet creation, launches background check loop
→ Reports check convergence result
```

**Example 8: Resuming a prepared step**
```
User: "/ManageImpStep prepare plans/myplan.md docs/myplan/README.md"
→ First non-done step has status: prepared
→ Sets it to in-progress, skips packet generation
→ Output: "Using pre-generated packet for step-id. Packet: plans/.../step-id.md"
```

## References

- [Design Document](references/Design.md) — Full design rationale and edge cases
