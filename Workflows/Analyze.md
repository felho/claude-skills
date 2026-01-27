# Analyze Prompt Workflow

Guide for analyzing existing prompts and suggesting improvements.

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
| Loops (`<loop>`) or conditions (if/then) | 3 - Control Flow |
| Task tool delegation, agent spawning | 4 - Delegate |
| Accepts prompt file as variable | 5 - Higher Order |
| Template section, creates new prompts | 6 - Template Metaprompt |
| Expertise section, self-updating | 7 - Self-Improving |

### 3. Audit Sections

Check which sections are present and their quality:

**Metadata:**
- [ ] Has YAML frontmatter?
- [ ] `description` is clear and concise?
- [ ] `allowed-tools` matches actual tool usage?
- [ ] `argument-hint` helps users understand input?

**Variables:**
- [ ] Uses SCREAMING_CASE naming?
- [ ] Dynamic variables use `$1`, `$2` or `$ARGUMENTS`?
- [ ] Static variables have sensible defaults?
- [ ] Variables are actually used in the workflow?

**Workflow:**
- [ ] Steps are numbered/ordered clearly?
- [ ] STOP conditions for missing required input?
- [ ] Loop blocks have descriptive names?
- [ ] Steps are actionable and specific?

**Instructions:**
- [ ] Rules are clear and necessary?
- [ ] Edge cases are covered?
- [ ] No redundancy with workflow?

**Report:**
- [ ] Output format is specified?
- [ ] Includes all necessary information?

### 4. Identify Issues

Common problems to flag:

**Structural Issues:**
- Missing required sections for the level
- Sections present but empty or vague
- Workflow steps that are too vague ("process the data")
- Variables defined but never used
- Tools used but not in `allowed-tools`

**Level Mismatches:**
- Using loops but at Level 2 (should be Level 3)
- Spawning agents but missing agent config variables
- Has Expertise section but no improve mechanism

**Quality Issues:**
- Variable names not descriptive
- No STOP condition for required variables
- Report section missing when output matters
- Instructions that belong in Workflow

### 5. Suggest Improvements

Organize recommendations by priority:

**High Priority** (functionality/correctness):
- Missing STOP conditions for required input
- Tools used without being in allowed-tools
- Workflow steps that can't be executed

**Medium Priority** (structure/clarity):
- Missing sections that would help
- Level upgrade if complexity warrants it
- Variable naming improvements

**Low Priority** (polish):
- Better description in metadata
- More detailed Report section
- Examples section for complex prompts

### 6. Provide Level Upgrade Path (if applicable)

If the prompt could benefit from a higher level:

```
Current: Level 2 (Workflow)
Suggested: Level 3 (Control Flow)

Why: The workflow has implicit conditionals that should be explicit.

Changes needed:
1. Add explicit if/then conditions
2. Consider loop blocks for repeated operations
3. Add early exit conditions
```

## Output

Provide analysis report:

```
## Prompt Analysis: <name>

**Current Level:** <N> - <Level Name>
**Sections Found:** <list>

### Strengths
- <what's good about this prompt>
- <another strength>

### Issues Found

**High Priority:**
- <critical issue>

**Medium Priority:**
- <structural issue>

**Low Priority:**
- <polish suggestion>

### Recommendations

1. <Specific actionable improvement>
2. <Another improvement>
3. <Optional enhancement>

### Level Recommendation
<Keep at Level N> OR <Consider upgrading to Level M because...>
```
