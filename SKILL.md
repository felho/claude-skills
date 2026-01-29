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
| **Prepare** | `/ManageImpStep prepare <plan> <design-doc> [step-id]` | `Workflows/Prepare.md` |
| **Check** | `/ManageImpStep check <packet>` | `Workflows/Check.md` |
| **Execute** | `/ManageImpStep execute <packet>` | `Workflows/Execute.md` |
| **Validate** | `/ManageImpStep validate <packet>` | `Workflows/Validate.md` |
| **Fix** | `/ManageImpStep fix <packet> [extra-feedback]` | `Workflows/Fix.md` |
| **Done** | `/ManageImpStep done <packet> [--no-commit]` | `Workflows/Done.md` |

## Workflow Flow

```
PREPARE → CHECK (optional) → EXECUTE → VALIDATE ──→ DONE
   │                                      ↑    ↓
   └── creates packet                     └── FIX
```

## Status Values

| State | HTML Comment |
|-------|--------------|
| Todo | `<!-- id: step-id -->` (no status) |
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

## References

- [Design Document](references/Design.md) — Full design rationale and edge cases
