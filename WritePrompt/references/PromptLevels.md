# The 7 Levels of Agentic Prompts

Detailed breakdown of each prompt level with characteristics, sections, and examples.

---

## Level 1: High Level Prompt

> Reusable, ad-hoc, static prompt.

**Characteristics:**
- Simple, single-purpose
- No dynamic input needed
- Can be run as-is

**Required Sections:**
- `# Title`
- High-level prompt text (the core instruction)

**Optional Sections:**
- `## Purpose`

**When to Use:**
- Quick utility commands
- Static information retrieval
- Simple one-shot tasks

**Example Structure:**
```markdown
# List All Tools

List all available tools detailed in your system prompt. Display them in bullet points.
```

---

## Level 2: Workflow Prompt

> Sequential workflow prompt with input, work, and output.

**Characteristics:**
- Defines clear steps to follow
- Has input variables and output format
- Linear execution flow

**Required Sections:**
- `## Workflow` — Numbered or bulleted steps

**Common Sections:**
- `Metadata` (YAML frontmatter)
- `## Variables` — Dynamic and static values
- `## Instructions` — Guidelines and constraints
- `## Report` — Output format specification
- `## Relevant Files`
- `## Codebase Structure`

**When to Use:**
- Multi-step processes
- Tasks with defined input/output
- Repeatable procedures

**Example Structure:**
```markdown
---
description: Gain understanding of the codebase
---

# Prime

Execute the `Workflow` and `Report` sections.

## Workflow

- Run `git ls-files` to list all files
- Read `README.md` for project overview

## Report

Summarize your understanding of the codebase.
```

---

## Level 3: Control Flow Prompt

> A prompt that runs conditions and/or loops in the workflow.

**Characteristics:**
- Contains conditional logic (if/then)
- May include loops or iterations
- Dynamic execution path

**Key Patterns:**
- `<loop-name>...</loop-name>` tags for iterations
- Conditional checks before actions
- Early exit conditions ("STOP immediately if...")

**When to Use:**
- Processing multiple items
- Conditional branching based on input
- Retry logic or polling

**Example Patterns:**
```markdown
## Workflow

- If no `PATH_TO_FILE` provided, STOP immediately and ask user
- For each item in the list, execute:
  <process-loop>
  - Extract data
  - Validate format
  - Save result
  </process-loop>
- After all items processed, generate summary
```

---

## Level 4: Delegate Prompt

> A prompt that delegates work to other agents (primary or subagents).

**Characteristics:**
- Uses Task tool to spawn agents
- Can run agents in parallel
- Coordinates multiple workers

**Key Sections:**
- `## Variables` with agent configuration (model, count, tools)
- Delegation blocks in workflow

**Agent Configuration Variables:**
```markdown
## Variables

PROMPT_REQUEST: $1
COUNT: $2
MODEL: sonnet (or opus for complex tasks)
```

**Delegation Patterns:**
```markdown
## Workflow

1. Parse input to understand task scope
2. Design agent prompts (self-contained, complete context)
3. Launch parallel agents using Task tool
4. Collect and synthesize results
```

**When to Use:**
- Parallel processing of independent tasks
- Work that benefits from multiple perspectives
- Long-running background operations

---

## Level 5: Higher Order Prompt

> Accept another reusable prompt (file) as input.

**Characteristics:**
- Takes a prompt file path as variable
- Provides consistent execution structure
- The inner prompt can be swapped

**Required:**
- `## Variables` with prompt file variable

**Example Structure:**
```markdown
## Variables

PATH_TO_PLAN: $ARGUMENTS

## Workflow

- If no `PATH_TO_PLAN` provided, STOP and ask user
- Read the plan at `PATH_TO_PLAN`
- Execute according to plan structure
```

**When to Use:**
- Build systems (plan → build pattern)
- Composable prompt chains
- Standardized execution of varying content

---

## Level 6: Template Metaprompt

> A prompt that creates new prompts in a specific format.

**Characteristics:**
- Output is a new prompt file
- Defines template structure
- May fetch documentation for context

**Required Sections:**
- `## Template` or `## Specified Format` — The output structure

**Key Patterns:**
```markdown
## Workflow

- Analyze the high-level request
- Fetch relevant documentation
- Create prompt following `Specified Format`
- Save to `.claude/commands/<name>.md`

## Specified Format

\`\`\`md
---
allowed-tools: <tools>
description: <description>
---

# <Title>

## Variables
<NAME>: $1

## Workflow
<steps>

## Report
<output format>
\`\`\`
```

**When to Use:**
- Prompt generators
- Standardizing prompt creation
- Building prompt libraries

---

## Level 7: Self-Improving Prompt

> A prompt that updates itself or is updated by another prompt with new information.

**Characteristics:**
- Contains `## Expertise` section
- Knowledge accumulates over time
- Often has companion "improve" prompt

**Required Sections:**
- `## Expertise` — Accumulated domain knowledge

**Pattern: Expert Trio**
```
expert_plan.md   → Plans work, reads expertise
expert_build.md  → Executes work, uses expertise
expert_improve.md → Analyzes results, UPDATES expertise sections
```

**Expertise Section Structure:**
```markdown
## Expertise

### Domain Knowledge
- Discovered patterns
- Best practices
- Architecture decisions

### Learned Standards
- File structures
- Naming conventions
- Security considerations
```

**When to Use:**
- Domain expert systems
- Learning from execution
- Evolving best practices

---

## Level Selection Guide

```
Need simple utility?           → Level 1
Need sequential steps?         → Level 2
Need conditions/loops?         → Level 3
Need parallel agents?          → Level 4
Need to accept prompt files?   → Level 5
Need to generate prompts?      → Level 6
Need accumulating knowledge?   → Level 7
```

Higher levels include capabilities of lower levels. Choose the minimum level that meets your needs.
