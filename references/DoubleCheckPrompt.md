# Double-Check Prompt: Structured Semantic Quality Verification

> **Reference document:** This defines the 7 semantic dimensions used by the Check workflow's `--double-check` mode (Phase B). The primary consumer is `Workflows/Check.md` Step 8B. It can also be used standalone as an agent prompt template (with placeholder substitution).

You are performing an independent semantic quality verification of a step implementation packet. This is a structured double-check — you evaluate the packet against 7 dimensions using a strict evidence-based rubric.

## Source Documents

- **Packet:** {PACKET_PATH}
- **Plan:** {PLAN_PATH}
- **Design document:** {DESIGN_DOC_PATH}

## Instructions

1. Read ALL three documents thoroughly (packet, plan, design doc). Do NOT use the `limit` parameter — read full files.
2. For each of the 7 dimensions below, evaluate the packet.
3. For each dimension:
   - **Quote specific evidence** from the packet that satisfies the dimension (or note its absence).
   - Cross-reference against the plan and design doc to verify completeness.
   - Score: **PASS** / **WEAK** / **MISSING**
4. If ANY dimension scores WEAK or MISSING:
   - Fix the packet by editing the file — add missing content from the plan and design doc.
   - Do NOT remove existing content. Do NOT duplicate existing content.
5. After all fixes, re-read the packet to verify your edits are consistent.

## 7 Semantic Dimensions

### 1. Technical Sufficiency

Does the packet specify all types, interfaces, configurations, and file paths needed for implementation?

**Check for:**
- Type definitions and interfaces mentioned in design doc
- Configuration values and their defaults
- File paths for files to create or modify
- Import paths and module structure
- Function signatures and return types

**Score:**
- **PASS** — All technical artifacts explicitly specified with enough detail to write code
- **WEAK** — Some artifacts mentioned but missing detail (e.g., type name without fields, file path without content structure)
- **MISSING** — Key technical artifacts from the design doc not mentioned in the packet

### 2. Error Handling Coverage

Does the packet specify all error cases with their messages and recovery paths?

**Check for:**
- Error conditions from the design doc for this step's domain
- Exact error messages (verbatim from design doc if specified)
- Recovery behavior for each error (retry, abort, fallback, user message)
- Error propagation paths (where errors are caught vs re-thrown)

**Score:**
- **PASS** — All error cases from design doc present with messages and recovery paths
- **WEAK** — Error cases listed but missing messages or recovery behavior
- **MISSING** — Error cases from design doc not addressed in packet

### 3. Edge Cases & Validation

Does the packet cover boundary conditions, validation rules, and concurrency concerns?

**Check for:**
- Input validation rules (what's valid, what's rejected, error on invalid)
- Boundary values (empty arrays, null, zero, max values)
- Concurrency or ordering concerns (if applicable to this step)
- State preconditions (what must be true before this step runs)

**Score:**
- **PASS** — Edge cases and validation rules explicitly enumerated
- **WEAK** — Some edge cases mentioned but incomplete coverage
- **MISSING** — No edge case or validation discussion despite design doc specifying them

### 4. Testability

Are test cases concrete enough to write actual test code without additional research?

**Check for:**
- Each test case has: input, expected output, and assertion
- Test cases cover happy path AND error paths
- Mock/stub requirements specified (what to mock, what values to return)
- Test file location specified

**Score:**
- **PASS** — Could write test code directly from the packet's test cases
- **WEAK** — Test scenarios described but missing concrete inputs, outputs, or assertions
- **MISSING** — Test section is vague or only lists categories without specific cases

### 5. Dependency Clarity

Are inter-step dependencies and external dependencies explicit?

**Check for:**
- Which prior steps must be complete (and what they provide)
- External packages/libraries needed (with version constraints if any)
- Shared resources (files, configs, databases) this step reads from or writes to
- Assumptions about environment or system state

**Score:**
- **PASS** — All dependencies listed with what they provide and how this step uses them
- **WEAK** — Dependencies listed but missing detail on what they provide
- **MISSING** — Dependencies not mentioned despite design doc showing them

### 6. Acceptance Criteria Clarity

Is each acceptance criterion specific, measurable, and binary (pass/fail)?

**Check for:**
- Each AC can be verified without subjective judgment
- ACs cover ALL deliverables mentioned in the step definition
- No vague criteria ("works correctly", "handles errors properly")
- Each AC maps to at least one test case

**Score:**
- **PASS** — All ACs are specific, measurable, binary, and cover all deliverables
- **WEAK** — Some ACs are vague or some deliverables lack corresponding ACs
- **MISSING** — ACs are mostly subjective or major deliverables have no AC

### 7. Implementation Completeness

Could an implementer start coding with ONLY this packet (no access to plan or design doc)?

**Check for:**
- All referenced sections from design doc are physically included (not just "see section X")
- All cross-references resolved to actual content
- No implicit knowledge required (the packet is self-contained)
- Implementation order or sequencing guidance if multiple files involved

**Score:**
- **PASS** — Packet is fully self-contained; implementer needs nothing else
- **WEAK** — Mostly self-contained but a few unresolved references or implicit assumptions
- **MISSING** — Packet relies heavily on external context not included

## Output Format

After evaluation, report results in this table:

```
## Double-Check Results

| # | Dimension | Score | Evidence / Gap |
|---|-----------|-------|----------------|
| 1 | Technical Sufficiency | PASS/WEAK/MISSING | [quote or gap description] |
| 2 | Error Handling Coverage | PASS/WEAK/MISSING | [quote or gap description] |
| 3 | Edge Cases & Validation | PASS/WEAK/MISSING | [quote or gap description] |
| 4 | Testability | PASS/WEAK/MISSING | [quote or gap description] |
| 5 | Dependency Clarity | PASS/WEAK/MISSING | [quote or gap description] |
| 6 | Acceptance Criteria Clarity | PASS/WEAK/MISSING | [quote or gap description] |
| 7 | Implementation Completeness | PASS/WEAK/MISSING | [quote or gap description] |

**Result:** ALL PASS / {N} dimensions need improvement
```

If ALL 7 dimensions score PASS: report "ALL PASS" and stop.

If ANY dimension scores WEAK or MISSING: fix the packet first, then report the table showing the ORIGINAL scores (before fixes) so the caller knows what was improved.
