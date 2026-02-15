# Review Prompt Workflow

Guide for reviewing existing prompts and suggesting improvements based on PromptSections.md standards. This workflow examines prompts without modifying them — use the Write workflow to apply changes.

## Input

`PROMPT_PATH`: Path to the prompt file to analyze, OR
`PROMPT_CONTENT`: The prompt content provided directly

## Workflow

### 1. Read the Prompt

- If path provided, read the file
- Parse YAML frontmatter (if present)
- Identify all sections used

### 2. Identify Current Level

Analyze the prompt to determine its level:

| Look for... | Indicates Level |
|-------------|-----------------|
| Just text, no structure | 1 - High Level |
| Workflow section with steps | 2 - Workflow |
| Loop blocks (`<*-loop>`) or conditionals (`If X → Y`) | 3 - Control Flow |
| Task tool delegation, agent spawning | 4 - Delegate |
| Accepts prompt file as variable | 5 - Higher Order |
| Template section, creates new prompts | 6 - Template Metaprompt |
| Expertise section, self-updating | 7 - Self-Improving |

### 3. Check Required Sections for Level (Top-Down)

Using the level identified in Step 2, verify that all required sections are present. Reference table (● = required, ○ = optional, - = not typical):

| Section | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
|---------|----|----|----|----|----|----|----|
| Metadata | ○ | ● | ● | ● | ● | ● | ● |
| Title | ● | ● | ● | ● | ● | ● | ● |
| Purpose | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| Variables | - | ● | ● | ● | ● | ● | ● |
| Instructions | - | ○ | ○ | ○ | ○ | ○ | ○ |
| Error Messages | - | - | ● | ● | ● | ○ | ● |
| Workflow | - | ● | ● | ● | ● | ● | ● |
| Report | - | ○ | ○ | ○ | ○ | ○ | ○ |
| Template | - | - | - | - | - | ● | ○ |
| Expertise | - | - | - | - | - | - | ● |
| Examples | ○ | ○ | ○ | ○ | ○ | ○ | ○ |

For each ● section at the identified level:
- [ ] Is the section present as a `##` heading with the standard name?
- If missing → flag as "Missing required section: {name} (required for Level {N})"

### 4. Semantic Placement Check (Bottom-Up)

Scan ALL content in the prompt and classify each block by its semantic role. Then verify it lives under the correct standard section heading.

**Semantic signatures to detect:**

| Semantic Role | Content Signals | Expected Section |
|---------------|----------------|------------------|
| **Rules/Constraints** | Bullet-point rules, "must", "never", "always", edge cases, guardrails | `## Instructions` |
| **Sequential Steps** | Numbered steps, "then", "next", "proceed to", action sequences | `## Workflow` |
| **Output Format** | Output templates with placeholders, "report:", "output:", result formatting | `## Report` |
| **Input Declarations** | `VAR: value`, `$1`, `$ARGUMENTS`, path declarations, config values | `## Variables` |
| **Usage Demos** | "Example:", user input → expected output pairs | `## Examples` |
| **File References** | File paths, "read this file", "modify this file" | `## Relevant Files` or `## Variables` |
| **Error Templates** | Error message strings, "Usage: ..." messages | `## Error Messages` |

For each content block identified:
- [ ] Does it live under the matching standard section heading?
- If the content is under a non-standard heading (e.g., `## Status Values` instead of `## Instructions`, `## Step 5: Report` instead of `## Report`) → flag as **"Misplaced content"**
- Provide: what the content is, where it currently lives, and where it should be

**Common misplacement patterns:**
- ❌ Rules/constraints scattered inside workflow steps → should be in `## Instructions`
- ❌ Output template inside a `## Step N` heading → should be in `## Report`
- ❌ File path declarations in a standalone section (e.g., `## Config`) → should be in `## Variables`
- ❌ STOP conditions only in `## Instructions` with no corresponding `## Error Messages` section (Level 3+)

### 5. Audit Section Syntax

For each present section, validate its syntax:

**Metadata (YAML Frontmatter):**
- [ ] Has YAML frontmatter with `---` delimiters?
- [ ] `description` is clear and concise (one line)?
- [ ] `allowed-tools` is comma-separated list matching actual tool usage?
- [ ] `argument-hint` uses correct bracket convention: `<required>` or `[optional]`?
- [ ] `model` (if present) is valid: `sonnet`, `opus`, or `haiku`?

**Variables:**
- [ ] Uses SCREAMING_CASE naming?
- [ ] Dynamic variables use `$1`, `$2` or `$ARGUMENTS`?
- [ ] Dynamic variables listed BEFORE static variables?
- [ ] Default syntax follows pattern: `VAR: $N or X if not provided`?
- [ ] Variables are actually used in the workflow?

**Workflow:**
- [ ] Exists as a single `## Workflow` section (not scattered as multiple `## Step N` top-level headings)?
- [ ] Steps are numbered sequentially (1, 2, 3...) WITHIN the `## Workflow` section?
- [ ] STOP conditions use correct syntax (see Control Flow Syntax below)?
- [ ] Loop blocks use correct syntax (see Control Flow Syntax below)?
- [ ] Steps are actionable and specific (not vague like "process the data")?

**Instructions:**
- [ ] Rules are written as bullet points?
- [ ] Edge cases are covered?
- [ ] No redundancy with workflow steps?

**Report:**
- [ ] Output format is specified with template?
- [ ] Includes all necessary information for the user?

### 6. Validate Control Flow Syntax (Level 3+)

**CRITICAL: These syntax patterns must be validated exactly.**

#### 6.1 STOP Conditions

**Correct syntax:**
```markdown
- If `VARIABLE` is empty → STOP and report: `"Usage: /command <arg>"`
- If file not found → STOP with "File not found: {path}" error
```

**Checklist:**
- [ ] Uses `→ STOP` arrow notation (not just "STOP")?
- [ ] Has associated error message in quotes?
- [ ] Error message includes context (variable name, path, etc.)?
- [ ] If Error Messages section exists, STOP references it?

**Common mistakes:**
- ❌ `STOP immediately` (missing arrow and message)
- ❌ `→ STOP` (missing error message)
- ✅ `→ STOP with "error message"` or `→ STOP and report: "message"`

#### 6.2 Loop Blocks

**Correct syntax:**
```markdown
3. For each item in collection:
   <descriptive-name-loop>
   - Step inside loop
   - Another step
   </descriptive-name-loop>
```

**Checklist for existing loop tags:**
- [ ] Opening and closing tags match exactly?
- [ ] Tag name ends with `-loop` suffix (distinguishes from named blocks)?
- [ ] Preceded by iteration context ("For each X...", "Process all Y...")?
- [ ] Contains bullet points for loop steps?

**Checklist for MISSING loop tags (scan entire Workflow section):**
- [ ] Search for iteration language patterns: `For each`, `For every`, `Process all`, `Iterate over`, `Loop through`
- [ ] For each occurrence, verify it has an associated `<*-loop>` tag
- [ ] **Check inside code blocks too** — pseudocode/decision logic in code blocks is still part of the workflow
- [ ] If iteration language exists without loop tag → flag as "Missing loop block"

**Iteration language detection patterns:**
```
For each X in Y:
For every X:
Process all X:
Iterate over X:
Loop through X:
```

**Common mistakes:**
- ❌ `<file-check>` (missing `-loop` suffix for loops)
- ❌ Loop without iteration context
- ❌ Mismatched tags: `<process-loop>...</processing-loop>`
- ❌ Iteration language in code block without loop tag (e.g., pseudocode with "For each file:")
- ✅ `<file-check-loop>` with "For each file:" context

#### 6.3 Named Blocks (Non-Loop)

**For grouping related content (NOT iteration):**
```markdown
<section-name>
- Related item 1
- Related item 2
</section-name>
```

**Checklist:**
- [ ] Does NOT end with `-loop` suffix (reserved for loops)?
- [ ] NOT preceded by iteration language ("For each...")?
- [ ] Opening and closing tags match?

**Distinction:**
- `<validation-criteria>` = Named block (grouping)
- `<validation-loop>` = Loop block (iteration)

#### 6.4 Conditional Statements

**Correct syntax:**
```markdown
- If condition → action
- If X is true → do Y
- If status: done → STOP with "Already done" error
```

**Checklist:**
- [ ] Uses `→` arrow notation for then-clause?
- [ ] Condition is clear and testable?
- [ ] Action is specific?

### 7. Identify Issues

Organize problems by severity:

**High Priority (functionality/correctness):**
- Missing STOP conditions for required variables
- STOP conditions without proper `→ STOP` syntax
- Loop blocks without `-loop` suffix (ambiguous)
- **Iteration language without loop tag** (including in code blocks)
- Mismatched opening/closing tags
- Tools used but not in `allowed-tools`
- Workflow steps that can't be executed
- **Misplaced content** — content semantically belongs to one section but lives under a different/non-standard heading (from Step 4)
- **Missing required section** for the identified level (from Step 3)

**Medium Priority (structure/clarity):**
- Missing optional-but-recommended sections for the level
- Variables not in correct order (dynamic → static)
- Default values not using standard syntax
- Loop blocks without iteration context
- Named blocks using `-loop` suffix incorrectly

**Low Priority (polish):**
- Description could be clearer
- Report section could be more detailed
- Missing Examples section for complex prompts
- argument-hint not using bracket convention

### 8. Suggest Improvements

For each issue found, provide:
1. **What's wrong** — specific problem
2. **Where** — line or section reference
3. **How to fix** — concrete correction

**Example 1:**
```
Issue: Loop block missing -loop suffix
Where: Step 7, <file-check> tag
Fix: Rename to <file-check-loop> and ensure preceded by "For each file..."
```

**Example 2:**
```
Issue: Iteration language without loop tag (in code block)
Where: Step 9, code block containing "For each file in git status:"
Fix: Convert pseudocode to proper loop block syntax:
     <git-status-check-loop>
     - Is it an expected deliverable? → OK
     - Is it in KNOWN_SIDE_EFFECTS? → OK
     - ...
     </git-status-check-loop>
```

### 9. Provide Level Upgrade Path (if applicable)

If the prompt could benefit from a higher level:

```
Current: Level 2 (Workflow)
Suggested: Level 3 (Control Flow)

Why: The workflow has implicit conditionals that should be explicit.

Changes needed:
1. Convert "if X, do Y" prose to "If X → Y" syntax
2. Add `-loop` suffix to iteration blocks
3. Add explicit STOP conditions with → syntax
```

## Output

Provide analysis report:

```
## Prompt Analysis: <name>

**Current Level:** <N> - <Level Name>
**Sections Found:** <list>

### Required Sections (Level <N>)

| Section | Required | Present | Status |
|---------|----------|---------|--------|
| <section> | ●/○ | ✅/❌ | <ok/missing> |

### Semantic Placement

| Content | Current Location | Expected Section | Status |
|---------|-----------------|-----------------|--------|
| <description> | <where it is> | <where it should be> | ✅/❌ |

### Syntax Compliance

| Pattern | Status | Notes |
|---------|--------|-------|
| STOP conditions | ✅/⚠️/❌ | <details> |
| Loop blocks | ✅/⚠️/❌ | <details> |
| Named blocks | ✅/⚠️/❌ | <details> |
| Conditionals | ✅/⚠️/❌ | <details> |
| Variable syntax | ✅/⚠️/❌ | <details> |

### Strengths
- <what's good about this prompt>
- <another strength>

### Issues Found

**High Priority:**
- <critical issue with fix>

**Medium Priority:**
- <structural issue with fix>

**Low Priority:**
- <polish suggestion>

### Recommendations

1. <Specific actionable improvement>
2. <Another improvement>
3. <Optional enhancement>

### Level Recommendation
<Keep at Level N> OR <Consider upgrading to Level M because...>
```
