# CraftPRD

A comprehensive Claude Code skill for the entire PRD (Product Requirements Document) lifecycle — from vague idea exploration through structured specification, multi-agent review, automated validation, and finding resolution.

## Why This Skill Exists

Writing a good PRD is hard. Writing a PRD that stays consistent as it grows past 500 lines is nearly impossible without tooling. CraftPRD addresses this by providing:

- **Progressive workflows** that match the PRD's maturity stage (idea → draft → structured spec)
- **Multi-agent review** with 7 specialized expert agents that each check a different dimension
- **Automated validators** that mechanically verify cross-references, type consistency, and structural completeness
- **Complexity detection** that tells you when your PRD has outgrown prose and needs restructuring into code

## How It Works

CraftPRD provides 7 workflows that cover the full PRD lifecycle:

### Phase 1: Exploration and Drafting

#### Explore (L2)
Start with a vague idea and develop it through structured questioning. Produces a one-pager that captures the problem, proposed solution, key decisions, and open questions.

```
User: "I want to build a task queue system"
→ Asks clarifying questions about scale, persistence, priorities
→ Produces a one-pager: problem statement, solution sketch, key decisions, open questions
```

#### Draft (L2)
Take the one-pager (or any rough notes) and expand it into a full PRD draft. Fills in sections systematically: problem statement, user stories, technical design, data model, API, etc.

### Phase 2: Deepening and Structuring

#### Deepen (L3)
Iteratively improve the PRD by asking probing questions about weak areas. After each iteration, checks complexity signals to determine if the PRD needs restructuring.

**Complexity signals include:**
- File length > 800 lines
- More than 5 TypeScript interfaces in markdown
- Same enum values in 3+ places
- More than 10 "see section X" cross-references
- State machine + sequence diagram in ASCII

#### Structure (L3)
When complexity signals fire, this workflow walks through "the usual suspects" — 13 areas that commonly need restructuring — and creates a migration plan to move technical contracts from prose to code.

**The Usual Suspects:**
1. Core Domain Model → TypeScript types
2. State Machine → XState or formal transition table
3. API Contract → Shared TypeScript + zod schemas
4. Communication Protocol → Existing framework or formal spec
5. Agent/LLM Interaction → AgentRunner abstraction
6. Invariants → Assertion functions
7. Error Handling → Error code enum with classification
8. Business Rules → Code for logic, prose for rationale
9. Security Model → Middleware + docs
10. Recovery/Resilience → Framework config + state machine events
11. Configuration → YAML schema with defaults
12. UI Specification → Stays in prose/design tool
13. Persistence Schema → SQL DDL or ORM schema

### Phase 3: Review and Validation

#### Review (L4)
The flagship workflow. Launches 7 specialized expert agents in parallel, each reviewing the PRD from a different perspective:

| Agent | Focus |
|-------|-------|
| **ApiContractChecker** | REST/WS API design, payload completeness, error responses |
| **ArchitecturalChecker** | Separation of concerns, coupling, scalability patterns |
| **BuildVsReuseAdvisor** | Custom implementations vs existing frameworks, representation drift |
| **CompletenessChecker** | Missing definitions, unspecified behavior, gaps, persistence |
| **CrossReferenceChecker** | Broken "see section X" links, field name consistency |
| **StateMachineChecker** | Reachability, transition completeness, recovery paths |
| **TypeConsistencyChecker** | Same types defined differently, cross-medium consistency |

Each agent must read all files before reporting findings, must quote evidence from the actual code, and follows a structured finding format with severity levels (Critical/Important/Minor).

#### Validate (planned)
Automated validation using deterministic rule-based validators (not LLM-based). Three validators check:

- **CrossReferenceIntegrity** — Import resolution, emit→event names, guards→functions, config references, command→ACK coverage, prose file references
- **StructuralCompleteness** — Transition required fields, side effects coverage for mutable fields, error path→error code coverage, terminal state identification
- **TypeConsistency** — No inline literal unions, branded type usage, schema-type alignment, cross-file field type consistency

### Phase 4: Resolution

#### ProcessFindings (L3)
After review, this workflow helps process the findings: triage by severity, group related findings, plan fixes, and track resolution progress.

## The Medium Guide

A key concept in CraftPRD is choosing the right medium for each piece of information:

| Medium | Best For | Enforces |
|--------|----------|----------|
| **Prose** | Decisions, rationale, flows, "why" | Nothing (human discipline) |
| **Code** (TypeScript) | Types, schemas, contracts, invariants | Compiler + runtime (zod) |
| **State machine** | Status transitions, guards, side effects | Formal verification |
| **Config** (YAML/JSON) | Tuneable parameters, feature flags | Schema validation |

The rule of thumb: if you deleted the design doc, could someone still implement the system correctly from the code specs alone? They should be able to — but they wouldn't understand WHY it was designed that way. The prose doc provides the "why."

## Directory Structure

```
CraftPRD/
├── SKILL.md                                    # Main skill definition (7 workflows)
├── README.md                                   # This file
├── Workflows/
│   ├── Explore.md                              # Idea → one-pager
│   ├── Draft.md                                # One-pager → full PRD
│   ├── Deepen.md                               # Iterative improvement with complexity checks
│   ├── Structure.md                            # Restructure prose into code specs
│   ├── Validate.md                             # Automated rule-based validation
│   ├── Review.md                               # Multi-agent expert review (7 agents)
│   └── ProcessFindings.md                      # Triage and resolve review findings
├── agents/                                     # Review agent prompts (used by Review workflow)
│   ├── ApiContractChecker.md
│   ├── ArchitecturalChecker.md
│   ├── BuildVsReuseAdvisor.md
│   ├── CompletenessChecker.md
│   ├── CrossReferenceChecker.md
│   ├── StateMachineChecker.md
│   └── TypeConsistencyChecker.md
├── validators/                                 # Deterministic validation rules
│   ├── CrossReferenceIntegrity.md
│   ├── StructuralCompleteness.md
│   └── TypeConsistency.md
├── references/
│   ├── ComplexitySignals.md                    # When a PRD needs restructuring
│   ├── MediumGuide.md                          # What goes where (prose vs code vs state machine)
│   ├── PRDProcessAnalysis.md                   # Review cycle metrics and analysis
│   ├── UsualSuspects.md                        # 13 areas to check when restructuring
│   ├── ValidateWorkflowDesign.md               # Design rationale for the Validate workflow
│   └── Templates/
│       ├── OnePager.md                         # One-pager template (Explore output)
│       └── StructuredSpec.md                   # Structured spec template (Structure output)
└── Tools/
    └── .gitkeep
```

## Usage Examples

### Example 1: Start from a vague idea

```
User: "I want to build a real-time collaboration tool"
→ /CraftPRD explore
→ Asks: "What type of collaboration? Documents, code, whiteboard?"
→ Asks: "How many concurrent users? What's the latency requirement?"
→ Produces one-pager with problem, solution sketch, 5 open questions
```

### Example 2: Review an existing PRD

```
User: "Review the Bob V1 PRD"
→ /CraftPRD review
→ Launches 7 expert agents in parallel (Opus model)
→ Each agent reads all files, produces findings with quoted evidence
→ Merges results: 12 Critical, 8 Important, 15 Minor findings
→ Presents sorted by severity with specific fix recommendations
```

### Example 3: PRD is getting too complex

```
User: "This PRD is getting unwieldy"
→ /CraftPRD structure
→ Scans for complexity signals: 950 lines, 8 inline interfaces, same enum in 4 places
→ Walks through the 13 "usual suspects"
→ Proposes migration plan: types.ts, state-machine.ts, protocol/, config/
→ Design doc shrinks to ~400 lines of decisions and rationale
```

### Example 4: Process review findings

```
User: "Go through the findings from the review"
→ /CraftPRD process-findings
→ Groups 35 findings by category
→ Triages: 5 need immediate fix, 8 are duplicates, 3 are false positives
→ Plans fixes in priority order, tracks resolution
```

## Review Agent Quality Controls

Based on real-world experience, the Review workflow includes several safeguards:

- **Opus model required** — Sonnet agents were found to skip tool use and hallucinate findings
- **Mandatory reading phase** — Agents must read all files before producing any findings
- **Zero-read guard** — Agents with 0 Read tool calls have all findings discarded
- **Evidence requirement** — Every finding must include a quoted code snippet from actual Read output
- **Severity calibration** — Critical = runtime bug/data loss, Important = implementation confusion, Minor = cosmetic

## Installation

This skill is part of the `~/.claude/skills/` directory and activates automatically when Claude Code detects PRD-related requests. No additional setup is required.

## When Does It Activate?

CraftPRD triggers on:
- "design product", "write PRD", "review PRD", "structure PRD"
- "validate PRD", "check consistency", "verify PRD", "run checks"
- "one-pager", "too complex", "complexity signals"
- "split PRD", "process findings", "fix findings", "go through findings"
