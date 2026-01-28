# Agentic Prompt Sections Reference

Complete guide to all available sections for building agentic prompts.

---

## Metadata (YAML Frontmatter)

Configuration block at the top of the prompt file.

```yaml
---
name: PromptName
description: What it does in one line
argument-hint: <required-arg> [optional-arg]
allowed-tools: Read, Write, Bash, Task
model: sonnet
---
```

**Bracket Convention:**
- `<arg>` = Required argument
- `[arg]` = Optional argument

**Fields:**
| Field | Purpose | Example |
|-------|---------|---------|
| `name` | Identifier for the prompt | `CreateImage` |
| `description` | Brief purpose (shown in help) | `Generate images via API` |
| `argument-hint` | Guide for user input | `<required> [optional]` |
| `allowed-tools` | Restrict available tools | `Read, Write, Bash` |
| `model` | Override default model | `sonnet`, `opus` |

---

## # Title

The main heading that names the prompt.

**Guidelines:**
- Use clear, action-oriented name
- Should communicate what the prompt does
- Match the filename (kebab-case → Title Case)

```markdown
# Create Image
# Load Context Bundle
# Claude Code Hook Expert Plan
```

---

## ## Purpose

Describes what the prompt accomplishes and when to use it.

**Guidelines:**
- Set context for the user
- Reference key sections (Workflow, Instructions)
- Keep it brief but complete

```markdown
## Purpose

Execute the `Workflow` and `Report` sections to understand the codebase,
then summarize your findings for the user.
```

---

## ## Variables

Defines inputs the prompt accepts.

**Variable Types:**

| Type | Syntax | Description |
|------|--------|-------------|
| Positional | `$1`, `$2`, `$3` | Indexed arguments |
| All args | `$ARGUMENTS` | Everything after command |
| Static | `VALUE: something` | Fixed values |

**Best Practices:**
- Dynamic variables first, static second
- Prefer `$1`, `$2` over `$ARGUMENTS` for multiple inputs
- Use SCREAMING_CASE for variable names
- Include defaults: `COUNT: $2 or 3 if not provided`

```markdown
## Variables

IMAGE_PROMPT: $1
COUNT: $2 or 3 if not provided
OUTPUT_DIR: agentic_output/images/
MODEL: google/nano-banana
```

---

## ## Instructions

Specific guidelines, rules, and constraints.

**Guidelines:**
- Write as bullet points
- Cover edge cases
- Define critical requirements
- Acts as guardrails for execution

```markdown
## Instructions

- If no prompt provided, STOP immediately and ask user
- Never read .env files directly
- Save all outputs to OUTPUT_DIR with timestamp
- Use exact values from variables, don't modify them
```

---

## ## Workflow

The core execution steps.

**Guidelines:**
- Use numbered list for sequence
- Include conditional logic where needed
- Define loop blocks with descriptive tags
- Be explicit about tool usage

```markdown
## Workflow

1. Validate all required variables are provided
2. Create output directory: `mkdir -p $OUTPUT_DIR`
3. For each item, execute the following:
   <process-loop>
   - Fetch data from API
   - Parse response
   - Save to file
   </process-loop>
4. Generate summary report
```

**Loop Block Pattern:**
```markdown
3. For each item in collection:
   <descriptive-name-loop>
   - Step inside loop
   - Another step
   </descriptive-name-loop>
```

**Requirements:**
- Tag name MUST end with `-loop` suffix
- MUST be preceded by iteration context ("For each X...", "Process all Y...")
- Opening and closing tags must match exactly

**Named Block Pattern (Non-Loop):**

For grouping related content WITHOUT iteration:
```markdown
<section-name>
- Related item 1
- Related item 2
</section-name>
```

**Requirements:**
- Tag name must NOT end with `-loop` (reserved for loops)
- NOT preceded by iteration language
- Use for grouping, categorization, or emphasis

**Distinction:**
| Type | Tag Example | Preceded By | Purpose |
|------|-------------|-------------|---------|
| Loop block | `<file-check-loop>` | "For each file:" | Iteration |
| Named block | `<validation-criteria>` | (nothing special) | Grouping |

**Conditional Syntax:**
```markdown
- If condition → action
- If X is empty → STOP with "error message"
- If status: done → skip this step
```

**Requirements:**
- Use `→` (arrow) notation for then-clause
- Condition must be clear and testable
- Action must be specific

**STOP Condition Pattern:**
```markdown
- If `VARIABLE` is empty → STOP and report: `"Usage: /command <arg>"`
- If file not found → STOP with "File not found: {path}" error
```

**Requirements:**
- Use `→ STOP` syntax (not just "STOP")
- Include error message in quotes
- Error message should include context (variable name, path, etc.)

---

## ## Error Messages

Define exact error messages for STOP conditions.

**Guidelines:**
- List all error cases with exact message text
- Use consistent format: `"Error: {description}"`
- Include placeholders for dynamic values: `{path}`, `{id}`
- Reference these from STOP conditions in Workflow

```markdown
## Error Messages (Use Exactly)

- Missing required argument: `"Usage: /command <required-arg>"`
- File not found: `"Error: File not found: {path}"`
- Invalid input: `"Error: {field} must be {constraint}. Got: {value}"`
```

---

## ## Relevant Files

Files the prompt needs to access.

**Guidelines:**
- List specific paths or patterns
- Explain why each file is relevant
- Group by purpose if many files

```markdown
## Relevant Files

- `README.md` — Project overview
- `src/config/*.ts` — Configuration files to modify
- `.claude/settings.json` — Hook configurations
```

---

## ## Codebase Structure

Directory layout context for the prompt.

**Guidelines:**
- Use tree-like format
- Include comments for key directories
- Show where outputs should go

```markdown
## Codebase Structure

```
project/
├── src/
│   ├── components/     # Vue components
│   └── utils/          # Helper functions
├── specs/              # Plan documents go here
└── .claude/
    └── commands/       # Custom slash commands
```
```

---

## ## Expertise

Accumulated domain knowledge (Level 7 prompts).

**Guidelines:**
- Organize by knowledge category
- Include discovered patterns
- Document best practices
- Update through companion "improve" prompt

```markdown
## Expertise

### Architecture Knowledge

**File Structure:**
- Hooks live in `.claude/hooks/`
- Configurations in `.claude/settings.json`

### Discovered Patterns

- Multiple hooks can target same event
- Use JSONL for streaming output
- Non-blocking errors: catch all, exit(0)

### Security Standards

- Always validate JSON input
- Use $CLAUDE_PROJECT_DIR for paths
- Never expose internal paths in errors
```

---

## ## Template

Output format for metaprompts (Level 6).

**Guidelines:**
- Wrap in code block
- Use placeholders: `<description>`
- Include all required sections
- Document what to replace

```markdown
## Template

\`\`\`md
---
allowed-tools: <tools based on task>
description: <one-line description>
argument-hint: <first-arg> [optional-arg]
---

# <Name Based on Purpose>

<Brief purpose statement>

## Variables

<VAR_NAME>: $1
<STATIC_VAR>: <fixed value>

## Workflow

<numbered steps>

## Report

<output format specification>
\`\`\`
```

---

## ## Examples

Concrete usage demonstrations.

**Guidelines:**
- Show 2-3 different use cases
- Include command and expected outcome
- Demonstrate edge cases if relevant

```markdown
## Examples

**Example 1: Basic usage**
```
User: /create-image "sunset over mountains"
→ Generates 3 images
→ Opens output folder
```

**Example 2: Custom count**
```
User: /create-image "abstract art" 5
→ Generates 5 images
→ Saves prompts alongside images
```
```

---

## ## Report

How to present results after execution.

**Guidelines:**
- Define structure and format
- Specify required information
- Can include markdown template

```markdown
## Report

After completion, provide:

```
✅ Task Complete

Files created: <count>
Output directory: <path>

Summary:
- <key result 1>
- <key result 2>
```
```

---

## Section Usage by Level

| Section | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
|---------|----|----|----|----|----|----|----|
| Metadata | ○ | ● | ● | ● | ● | ● | ● |
| Title | ● | ● | ● | ● | ● | ● | ● |
| Purpose | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| Variables | - | ● | ● | ● | ● | ● | ● |
| Instructions | - | ○ | ○ | ○ | ○ | ○ | ○ |
| Error Messages | - | - | ● | ● | ● | ○ | ● |
| Workflow | - | ● | ● | ● | ● | ● | ● |
| Relevant Files | - | ○ | ○ | ○ | ○ | ○ | ○ |
| Codebase Structure | - | ○ | ○ | ○ | ○ | ○ | ○ |
| Template | - | - | - | - | - | ● | ○ |
| Expertise | - | - | - | - | - | - | ● |
| Examples | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| Report | - | ○ | ○ | ○ | ○ | ○ | ○ |

● = Required/Common | ○ = Optional | - = Not typical

**Note:** Error Messages section is recommended for Level 3+ prompts that have STOP conditions.
