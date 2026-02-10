# Check Quality Analysis: From Trust Gap to Deterministic Validators

> **Date:** 2026-02-09 — 2026-02-10
> **Session:** `fecc7651-9aa2-4412-8b64-854dde89111d` (analysis), then implementation session
> **Trigger:** Auto-check convergence followed by manual check that appeared to find new issues

## 1. The Problem

The ManageImpStep skill has a quality loop: Prepare creates a packet, Check verifies it against source documents. The `--auto-check` flag runs Check in a background loop until zero findings — then the packet is marked ready.

The trust question arose from a specific incident:

**Session A** (`b1aeab7e-362d-41e1-be92-13bd38b974dc`): Ran `Prepare --auto-check` for the Bob V1 PRD Review Pipeline's first step (`foundation/monorepo-setup`). The auto-check agent iterated 4 times and converged with zero findings.

**Session B** (`3e80b01d-3aa5-4fc7-a51a-4bb5024e9ffb`): Ran manual `/ManageImpStep check` on the same packet. The check agent reported 10 "Potential Issues Found."

This raised the question: **Can we trust auto-check convergence?** If a manual check immediately finds 10 issues after auto-check says "clean," the system cannot be scaled to orchestrated execution.

> "It is paramount that we can trust these steps. If you cannot trust them, then we can't scale them. The idea is that over time I don't want to run them manually, but I would like to run them with some kind of orchestration process."

## 2. The Analysis

### What Actually Happened

The 10 "Potential Issues" from the manual check were **all verified as already present** in the packet (all 10 marked "Found" with line numbers). The manual check's conclusion was: "Packet is complete. All potential issues verified as present in packet."

So the auto-check had worked correctly — the packet was indeed complete. The confusing output was the Check workflow's methodology: it first generates hypotheses ("Potential Issues Found"), then verifies each one. The 10 items were hypotheses, not confirmed gaps.

### The Real Finding: Superficial Reading

The critical discovery was not about auto-check convergence, but about **how the check agent read the source documents**:

- The design doc is ~2237 lines long
- The manual check agent read it with `limit: 200` — only the first 200 lines
- The Check.md workflow explicitly says: "Read the **ENTIRE** design document" and "Take your time. Read carefully. Don't skim."

The agent ignored the instruction and skimmed. This means the manual check provided **false confidence** — it verified the packet was complete against 9% of the design doc.

By contrast, the auto-check agent in Session A used Grep to find relevant sections in the design doc and read them targeted — a better strategy, though still not guaranteeing full coverage.

### Non-Deterministic Finding Generation

The two sessions generated completely different "Potential Issues" lists:

| Auto-check (Session A, final iter) | Manual check (Session B) |
|-------------------------------------|--------------------------|
| 6 structured checklist items | 10 ad-hoc questions |
| Technical specs focus | Plan-vs-packet diff focus |
| Used Grep for targeted design doc reads | Used Read with `limit: 200` |

This is inherent to LLM-based checking: every run examines the packet from a different angle. This diversity is actually valuable — the problem is when a run **skips important angles entirely** because it didn't read the source material.

## 3. The Architectural Insight

The analysis revealed that packet quality validation was **entirely LLM-based**, making it:
- Non-deterministic (different runs, different findings)
- Unverifiable (no way to know if the agent actually read the full design doc)
- Non-structural (format and coverage checks mixed with semantic judgment)

The key insight: **split validation into two layers**.

```
┌─────────────────────────────────────────────┐
│  Layer 1: Deterministic (Python validators) │
│  - Packet structure (frontmatter, sections) │
│  - Plan checkbox coverage                   │
│  - Design ref resolution                    │
│  - Acceptance criteria count                │
│  Guaranteed: same input → same output       │
├─────────────────────────────────────────────┤
│  Layer 2: Semantic (LLM check)              │
│  - "Is there enough context?"               │
│  - "Are edge cases covered?"                │
│  - "Would an implementer have everything?"  │
│  Inherently non-deterministic, but valuable │
└─────────────────────────────────────────────┘
```

Layer 1 handles the things that **can be checked deterministically** — format compliance, checkbox coverage, reference resolution. These should never be LLM judgment calls.

Layer 2 remains for things that **require understanding** — semantic completeness, implementation sufficiency. The LLM is still the right tool here, but it no longer needs to waste attention on structural concerns.

## 4. The Solution: Self-Validation Hooks

The WritePrompt skill's Harden workflow provided the implementation pattern: Python validator scripts embedded as hooks in workflow YAML frontmatter. These fire automatically during workflow execution, creating a closed-loop self-correction cycle.

### Architecture

```
Prepare/Check workflow
    │
    ├─ [PostToolUse: Write|Edit] → packet-structure-validator.py
    │   Fires after every file write/edit.
    │   Validates format immediately.
    │   If block → agent gets error → fixes → retries.
    │
    └─ [Stop] → packet-coverage-validator.py
        Fires when workflow finishes.
        Validates completeness against plan.
        If block → agent gets error → fixes → Stop fires again.
```

### What the Validators Check

**Structure validator** (PostToolUse, per-file):
1. YAML frontmatter has 4 required fields: `step`, `plan`, `design-doc`, `created`
2. All 8 required H2 sections exist
3. No required section is empty
4. Frontmatter `step` field matches file path pattern

**Coverage validator** (Stop, end-of-workflow):
1. All plan step checkboxes appear in packet Step Definition (fuzzy match)
2. Plan `**Design ref:**` entries resolve to content in packet's "From Design Document" section
3. Acceptance Criteria count >= plan checkbox count
4. Test Cases section has at least one item

### Design Decisions

- **stdlib-only** (`dependencies = []`) — zero install time, no dependency management
- **JSON stdout protocol** — `{}` to pass, `{"decision": "block", "reason": "..."}` to fail
- **`$HOME` in hook commands** — reliable shell expansion
- **Fuzzy checkbox matching** — plan checkboxes are often abbreviated; packet extends them
- **PostToolUse for structure, Stop for coverage** — immediate feedback for format, final gate for completeness
- **Namespaced location** — `~/.claude/hooks/validators/ManageImpStep/` communicates skill ownership

### Verification Results

- Structure validator: **58/58 pass** on all real packets (Bob + wbc-monthly-close projects)
- Coverage validator: pass on well-formed packets, correctly flags gaps in older packets
- Gating: non-packet files, findings files, empty input → silent pass
- Fail case: broken packet → precise block response with actionable errors

## 5. What Remains Unsolved

The validators address the deterministic layer. Four concerns remain that need different approaches:

### 5.1 Design Doc Reading Depth Enforcement

**Problem:** The LLM check agent can ignore "Read the ENTIRE document" and use `limit` parameter to skim. This was the actual failure mode observed in Session B.

**Possible approach:** PreToolUse hook on Read that detects when the design doc path is being read with a `limit` parameter, and warns/blocks. The challenge is knowing which file is "the design doc" at hook time — the hook would need to match against frontmatter's `design-doc` field.

**Difficulty:** Medium. The hook can detect `limit` on Read calls to `*design*.md` files, but cannot force the agent to actually process the content it reads.

### 5.2 Semantic Quality

**Problem:** "Is there enough context for implementation?" is inherently subjective and requires domain understanding.

**Current mitigation:** The LLM check (Check.md workflow) handles this. The auto-check loop runs up to 10 iterations.

**Possible improvement:** A double-check schema where two independent agents check the same packet and findings are compared. Divergence beyond a threshold triggers human review. This trades cost for reliability.

### 5.3 Cross-Step Consistency ✅ SOLVED

**Solved:** 2026-02-10

**Problem:** Validators check one packet against one plan step. They cannot detect inconsistencies across packets (e.g., Step B references a type defined in Step A's packet, but Step A's packet defines it differently).

**Solution:** `cross-step-consistency-validator.py` — a single script with two modes:

- **Structure mode** (Stop hook on Prepare/Check): Discovers ALL sibling packets via plan path, builds full dependency graph, runs 7 checks: dependency existence, circular deps, frontmatter consistency, step ID validity, orphan detection, status constraints (at most one in-progress, deps done before execution), duplicate packets.
- **Readiness mode** (Stop hook on Execute, `--mode readiness`): Lightweight check that target packet's dependencies are all done in the plan, plus multiple-in-progress guard.

Dependency parsing handles real-world formats: `Step N.M (slug)` parenthesized and `Step N.M: slug` colon format. Tested against 58 wbc-monthly-close packets (clean pass on revolut-support, correctly detected real circular dep in personal-finance).

### 5.4 Auto-Check Convergence Failure

**Problem:** If auto-check runs 10 iterations without converging, the packet is still marked `prepared`. There is no signal to the orchestrator that quality is uncertain.

**Possible approach:** The Stop hook could check iteration count from the agent's output or from a metadata file. If iterations >= threshold, block with a warning requiring human review.

**Alternative:** A confidence metadata field in the packet frontmatter:
```yaml
check-confidence: converged  # converged | max-iterations | unchecked
check-iterations: 4
```

## 6. Lessons Learned

1. **LLM instructions are suggestions, not guarantees.** "Read the ENTIRE document" can be ignored. Deterministic enforcement requires hooks/validators, not prose instructions.

2. **The medium matters.** A numbered finding list ("10 Potential Issues Found") creates alarm even when all 10 are verified as present. The Check workflow's methodology was correct but its output format was misleading.

3. **Separate what you can measure from what you must judge.** Structural and coverage checks are measurable — make them deterministic. Semantic quality requires judgment — keep it LLM-based but don't mix it with structural concerns.

4. **Diversity in checking is valuable, not a bug.** Different LLM runs examining the packet from different angles increases coverage. The problem is ensuring each run has adequate source material access, not that they produce different findings.

5. **Trust requires layers.** For orchestrated execution, the confidence chain should be: deterministic validators pass (hard requirement) → LLM check converges (soft requirement) → confidence metadata recorded (observability).
