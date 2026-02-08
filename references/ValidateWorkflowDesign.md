# Validate Workflow — Design Notes

> Session notes from the design discussion about adding PRD code validation to CraftPRD.

## Motivation

The CraftPRD lifecycle moves PRDs from prose to a mix of prose + code (via the Structure workflow). After structuring, code lives in actual `.ts` files (`domain/types.ts`, `domain/state-machine.ts`, `protocol/commands.ts`, etc.) and the design doc references them.

Two questions prompted this design:

1. **Validation:** Can we verify that the code artifacts within a structured PRD are internally consistent? When someone modifies a `.ts` contract file, can we run a command that proves correctness?
2. **Implementation leverage:** When building the actual application, how do we use these code artifacts? (Answered naturally: they're already `.ts` files — the implementation imports them directly.)

## Key Insight: Structure Already Extracts Code

The Structure workflow (Phase 4: EXECUTE) already:
- Extracts code from prose into actual `.ts` files
- Updates the PRD to reference those files
- Creates a target directory structure (see MediumGuide.md)

**After Structure runs, there is no code in the markdown.** The PRD is prose-only with `"See domain/state-machine.ts"` references. Code lives in real TypeScript files.

This means the Validate workflow does NOT need to:
- Parse markdown for code blocks
- Extract code from prose
- Use annotations to identify code sections

The `.ts` files are already there. Validate works with them directly.

## What Validate Actually Does

The Validate workflow is the **test suite for the PRD's code artifacts**. Analogy: Structure creates the "production code" (types, state machines, contracts), Validate creates the "tests" for that code.

### Layer 1: Type Checking (`tsc --noEmit`)

Run the TypeScript compiler on the extracted contract files. This catches:
- Types referencing non-existent types
- Payload types not matching domain model
- State machine using undefined states
- Any cross-file type inconsistency

This is "free" validation — just running the compiler on files that already exist.

### Layer 2: Semantic Validation (Custom Rules)

Beyond what `tsc` can check, there are semantic properties that matter for PRD correctness:

**State Machine rules:**
- No dead states (can enter but never leave)
- No unreachable states (defined but can't reach from initial state)
- Every state has at least one outgoing transition (except terminal states)
- Initial state is defined
- Terminal states are explicitly marked

**Command/Handler Map rules:**
- Every command has a corresponding handler
- No orphan handlers (handler without a command)
- Payload types match between command definition and handler expectation

**Domain Model rules:**
- No circular required dependencies
- Referenced types exist (tsc catches this too, but custom rules give better error messages)

**API Contract rules:**
- Every endpoint has request + response types defined
- Error types are defined for each endpoint
- Discriminated unions are exhaustive

**Why custom rules matter even though we have tsc:**
- `tsc` checks type compatibility. Custom rules check **semantic completeness**.
- Example: `tsc` won't tell you a state machine has dead states — the types are valid, the design is flawed.
- LLM-based review (the Review workflow) can catch these too, but non-deterministically. Custom rules catch them every time.

**Why custom rules matter even though we have the Review workflow:**
- The Review workflow uses LLM agents — non-deterministic, expensive, slow
- Custom rules are deterministic, instant, free to run
- They serve as a fast feedback loop: change code → run validate → see issues immediately
- Think of it as: Review = code review by humans, Validate = CI checks

### Layer 3: Cross-Medium Consistency (stretch goal)

Check that the prose references to code files are still valid:
- File paths in `"See domain/state-machine.ts"` references actually exist
- Section headings referenced in prose haven't been renamed in code comments

## Proposed Output Structure

After the Validate workflow runs on a structured PRD:

```
bob-v1/
├── docs/
│   └── design.md                  ← prose (already exists after Structure)
├── packages/
│   ├── domain/
│   │   ├── types.ts               ← already exists after Structure
│   │   ├── state-machine.ts       ← already exists after Structure
│   │   └── invariants.ts          ← already exists after Structure
│   └── protocol/
│       ├── commands.ts            ← already exists after Structure
│       └── errors.ts              ← already exists after Structure
├── validate/                      ← NEW: created by Validate workflow
│   ├── tsconfig.json              ← minimal config for type checking
│   ├── validate.ts                ← entry point: npx tsx validate/validate.ts
│   └── rules/
│       ├── state-machine.test.ts  ← completeness, reachability, dead states
│       ├── command-coverage.test.ts ← command-handler pairing
│       └── invariants.test.ts     ← domain invariant checks
```

The user can then run:
```bash
npx tsx validate/validate.ts
```

This command:
1. Runs `tsc --noEmit` on the contract files
2. Runs all semantic rules
3. Reports results (pass/fail with details)

## Rule Generation: Hybrid Approach (Templates + LLM)

**For known patterns** (state machine, command map, domain model): use pre-written, tested templates. The Validate workflow maps the PRD's contract files to the appropriate template and parameterizes it.

**For PRD-specific invariants:** the LLM generates custom rules during the workflow. These are generated once, committed, and then run deterministically.

Key distinction: the LLM is involved only during **setup** (when the Validate workflow runs). After that, `npx tsx validate/validate.ts` is purely deterministic — no LLM needed.

## Workflow Design Sketch

```
Validate workflow (L3, agentic)

Input: path to a structured PRD project (post-Structure)
Precondition: Structure workflow has already run (code files exist)

Phase 1: DISCOVER
  - Read the project directory
  - Identify which contract files exist
  - Classify each file by pattern (state machine, domain model, API contract, etc.)

Phase 2: GENERATE
  - For each identified pattern:
    - Select the appropriate template
    - Generate the test/rule file, parameterized to this PRD's specific types
  - Ask LLM: "Are there PRD-specific invariants that should be validated?"
    - If yes, generate custom rule files
  - Create validate.ts entry point
  - Create tsconfig.json

Phase 3: RUN
  - Execute tsc --noEmit
  - Execute validate.ts
  - Report results

Phase 4: FIX (if issues found)
  - Present each issue to the user
  - Suggest fixes (in the contract files, not in the tests)
  - User decides: apply / modify / skip
```

## Impact on Existing Workflows

### Structure workflow
No changes needed. It already extracts code to `.ts` files. The Validate workflow is a natural next step after Structure completes.

### Review workflow
Complementary, not competing. Review = non-deterministic expert analysis (finds design issues, trade-off concerns, missing considerations). Validate = deterministic consistency checks (finds mechanical errors in the code artifacts). Run both, for different purposes.

### Lifecycle position
```
Explore → Draft → Deepen → Structure → Validate → Review → ProcessFindings
                                          ↑ NEW
```

Validate sits between Structure and Review: after code is extracted but before expert review. This way, Review agents don't waste time on issues that a compiler could have caught.

## Implementation Leverage (second original question)

Since Structure already creates real `.ts` files, the implementation leverage question answers itself:

1. **The contract files ARE the project's type definitions.** The implementation imports directly from `packages/domain/` and `packages/protocol/`.
2. **The compiler enforces conformance.** If the implementation doesn't match the PRD's types, it won't compile.
3. **Validate's test suite transfers.** The semantic rules (state machine completeness, command coverage) become the project's contract tests — run them in CI.

This is **schema-first development** applied to PRDs: the PRD defines the types, the implementation conforms to them, and the Validate suite ensures the types themselves are coherent.

## Open Questions

1. **Template library scope:** How many pattern templates do we need for V1? Proposed minimum: state machine completeness + command-handler coverage. Others can be added later.
2. **Where do templates live?** Options: CraftPRD `references/templates/validate/` or a separate directory.
3. **Dependency management:** The validate directory needs `tsx` and `typescript`. Should we generate a `package.json`? Or assume these are globally available?
4. **Incremental validation:** If only one contract file changes, can we run just the relevant rules? Or always run everything? (For V1: run everything. It should be fast.)
