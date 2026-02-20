# ManageImpStep

A Claude Code skill that drives multi-step implementation plan execution with focused context packets. It breaks down large implementation plans into individual step packets, each containing everything needed to implement that step — then guides you through a structured prepare → check → execute → validate → fix → done lifecycle.

## Why This Skill Exists

Large implementation plans (10+ steps across multiple phases) present a fundamental problem: by the time you're implementing step 7, the context window is full of steps 1-6 and there's no room for the actual design document details. ManageImpStep solves this by:

1. **Extracting focused packets** — Each step gets its own file with all relevant context from the plan AND design document
2. **Enforcing a lifecycle** — Every step goes through prepare → execute → validate → done, preventing half-finished work
3. **Supporting parallel preparation** — Multiple steps can be prepared ahead of time with `--ahead N`
4. **Enabling cross-session work** — Packets and findings persist on disk, so you can stop and resume across Claude Code sessions

## How It Works

### The Step Lifecycle

```
todo → prepared → in-progress → done
                      ↕
              validate ↔ fix
```

1. **Prepare** — Creates a context packet from the plan + design doc
2. **Check** — Verifies the packet has all information needed for implementation
3. **Execute** — Implements the step following the packet's instructions (TDD)
4. **Validate** — Verifies the implementation meets acceptance criteria
5. **Fix** — Targeted fixes for failed validation criteria
6. **Done** — Marks the step complete in the plan

### Plan Structure

Implementation plans use HTML comments for step identification and status tracking:

```markdown
## Phase: Data Setup
<!-- id: data-setup -->

### Create directory structure
<!-- id: create-dirs -->
- [ ] Create `data/` directory with subdirectories
- [ ] Add `.gitkeep` files

### Export spreadsheet data
<!-- id: export-sheets status: in-progress -->
- [ ] Export transactions CSV
- [ ] Export categories CSV
```

Full step IDs use the format `{phase-id}/{step-id}`, e.g., `data-setup/create-dirs`.

### Packet Structure

Each packet is a standalone Markdown file with YAML frontmatter:

```yaml
---
step: data-setup/export-sheets
plan: plans/personal-finance/personal-finance.md
design-doc: docs/personal-finance/README.md
created: 2026-01-27T10:00:00Z
check-confidence: double-checked
test-strategy: tdd
---
```

Followed by sections: Overview, Step Definition, From Design Document, From Plan, Dependencies, Test Cases, Acceptance Criteria, and Implementation Notes.

Packets are stored alongside the plan:

```
plans/
└── personal-finance/
    ├── personal-finance.md              # The plan
    └── personal-finance-steps/          # Generated packets
        ├── data-setup/
        │   ├── create-dirs.md
        │   └── export-sheets.md
        └── cli-foundation/
            ├── app-skeleton.md
            └── config-loading.md
```

## Workflows

### prepare

Creates a focused context packet for the next (or specified) step.

```
/ManageImpStep prepare plans/myplan.md docs/design.md
/ManageImpStep prepare plans/myplan.md docs/design.md cli-foundation/config-loading
/ManageImpStep prepare plans/myplan.md docs/design.md --ahead 3 --auto-check
```

**Key features:**
- **Auto-selection** — Without a step ID, picks the next todo step automatically
- **Early claim** — Sets status to `in-progress` immediately to prevent parallel conflicts
- **`--ahead N`** — Prepares N steps in parallel using Task agents
- **`--auto-check`** — Runs an automated check loop after preparation (up to 3 double-check cycles)

**Scope guardrail:** Prepare creates ONLY the packet documentation. It does NOT implement anything.

### check

Verifies packet completeness against the source documents.

```
/ManageImpStep check plans/myplan-steps/phase/step.md
```

Re-reads the plan and design doc, finds everything missing from the packet (edge cases, error messages, validation rules, test scenarios), and adds it. Idempotent — running it multiple times won't duplicate content.

### execute

Implements the step following the packet's instructions.

```
/ManageImpStep execute plans/myplan-steps/phase/step.md
```

Follows TDD: writes tests first, then implementation. Does not add features beyond what the packet specifies.

### validate

Verifies the implementation meets all acceptance criteria.

```
/ManageImpStep validate plans/myplan-steps/phase/step.md
/ManageImpStep validate plans/myplan-steps/phase/step.md --agents 3
```

**Single-agent mode** (default): Runs all checks inline — type check, tests, file existence, acceptance criteria, git status audit.

**Multi-agent mode** (`--agents N`): Launches N parallel validation agents that independently verify the implementation. Results are merged using a union strategy — if ANY agent finds an issue, it counts.

**Git status audit**: Checks that only expected deliverables appear in `git status`. Unexpected files trigger a user prompt.

**Findings file**: On failure, writes a structured `.findings.md` file that the fix workflow reads:

```
plans/myplan-steps/phase/step.findings.md
```

### fix

Applies targeted fixes for failed validation criteria.

```
/ManageImpStep fix plans/myplan-steps/phase/step.md
/ManageImpStep fix plans/myplan-steps/phase/step.md "use batch-meta.json for criterion 12"
```

Reads the findings file, applies TDD fixes for each failed criterion. Does NOT fix unrelated issues or improve passing code.

### done

Marks the step complete in the plan file.

```
/ManageImpStep done plans/myplan-steps/phase/step.md
/ManageImpStep done plans/myplan-steps/phase/step.md --commit
```

Checks all task checkboxes and changes `status: in-progress` to `status: done`. With `--commit`, also creates a git commit.

## The Validate → Fix Loop

```
execute → validate ──→ done
             ↑    ↓
             └── fix
```

This loop can repeat multiple times. Each iteration:
1. Validate produces a fresh findings file (complete snapshot, not append)
2. Fix reads findings, applies targeted corrections
3. Validate re-checks everything from scratch
4. When all criteria pass, findings file is deleted

## Check Confidence Metadata

After auto-check, the packet frontmatter records quality:

| Value | Meaning |
|-------|---------|
| `double-checked` | All 7 semantic dimensions PASS. Highest confidence. |
| `max-double-checks` | 3 cycles exhausted. Improved but may have gaps. |
| `converged` | Freeform check loop reached zero findings. |
| `max-iterations` | 10 iterations without converging. Quality uncertain. |
| `unchecked` | No auto-check ran. |

A Stop hook blocks execution when `check-confidence: max-iterations`, recommending manual review.

## Scope Guardrails

Each workflow has explicit boundaries to prevent common scope creep:

| Workflow | Can Modify |
|----------|------------|
| **Prepare** | Only the packet `.md` file + plan status |
| **Check** | Only the packet `.md` file |
| **Execute** | Implementation files (code, config, tests) |
| **Validate** | Only findings file (`*.findings.md`) |
| **Fix** | Implementation files (targeted to failed criteria) |
| **Done** | Only the plan `.md` file |

## Directory Structure

```
ManageImpStep/
├── SKILL.md                            # Main skill definition with workflow routing
├── README.md                           # This file
├── Workflows/
│   ├── Prepare.md                      # Create context packet
│   ├── Check.md                        # Verify packet completeness
│   ├── Execute.md                      # Implement step (TDD)
│   ├── Validate.md                     # Verify implementation
│   ├── Fix.md                          # Targeted fixes for failures
│   └── Done.md                         # Mark step complete
├── references/
│   ├── Design.md                       # Full design rationale and decisions
│   ├── CheckQualityAnalysis.md         # Analysis of check quality patterns
│   └── DoubleCheckPrompt.md            # 7-dimension structured double-check
└── Tools/
    └── .gitkeep
```

## Usage Examples

### Example 1: Start implementing a plan

```
User: "/ManageImpStep prepare plans/personal-finance/personal-finance.md docs/personal-finance/README.md"
→ Finds next todo step: data-setup/create-dirs
→ Reads plan + design doc thoroughly
→ Creates packet at plans/personal-finance/personal-finance-steps/data-setup/create-dirs.md
→ Sets step status to in-progress
→ Reports: "Step prepared. Run /ManageImpStep execute <packet> to implement."
```

### Example 2: Prepare multiple steps ahead

```
User: "/ManageImpStep prepare plans/myplan.md docs/design.md --ahead 3 --auto-check"
→ Finds next 3 todo steps
→ Launches 3 parallel Task agents, each preparing one step
→ Each agent runs a check loop until the packet is clean
→ Reports: 3 packets created, all double-checked
```

### Example 3: Validate and fix cycle

```
User: "/ManageImpStep validate plans/myplan-steps/phase/step.md --agents 3"
→ Launches 3 parallel validation agents
→ Agent 1: all pass. Agent 2: criterion #4 fails. Agent 3: all pass.
→ Merged result: FAIL (1/3 agents found issue with criterion #4)
→ Writes findings file

User: "/ManageImpStep fix plans/myplan-steps/phase/step.md"
→ Reads findings: criterion #4 failed
→ Writes test for the gap, implements fix
→ Reports: "Fix applied. Re-run validate."

User: "/ManageImpStep validate plans/myplan-steps/phase/step.md"
→ All criteria pass, findings file deleted
→ Reports: "PASS. Run /ManageImpStep done <packet> to mark complete."
```

### Example 4: Cross-session workflow

```
# Session 1: Prepare and start executing
/ManageImpStep prepare plans/myplan.md docs/design.md --auto-check
/ManageImpStep execute <packet>
# (session ends mid-implementation)

# Session 2: Continue where you left off
/ManageImpStep prepare plans/myplan.md docs/design.md
# → Detects in-progress step, reports packet path
/ManageImpStep execute <packet>
# → Continues implementation
/ManageImpStep validate <packet>
/ManageImpStep done <packet> --commit
```

## Installation

This skill is part of the `~/.claude/skills/` directory and activates automatically when Claude Code detects implementation step management requests. No additional setup is required.

## When Does It Activate?

ManageImpStep triggers on:
- "step prepare", "step check", "step execute", "step validate", "step fix", "step done"
- "implementation workflow", "plan packet"
- Any `/ManageImpStep` command invocation
