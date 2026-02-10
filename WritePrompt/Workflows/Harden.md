# Harden Prompt Workflow

Guide for adding self-validation hooks to existing prompts, agents, or commands. Transforms an open-loop prompt (hope it works) into a closed-loop system (deterministic validation after every relevant operation).

> **Reference:** [SelfValidationHooks.md](../references/SelfValidationHooks.md) — hook syntax, JSON protocol, validator template, design patterns.

## Input

`TARGET_PATH`: Path to the existing prompt/agent/command file to harden.

## Workflow

### 1. Load Reference Knowledge

- Read `references/SelfValidationHooks.md` (relative to the WritePrompt skill directory)
- This contains the YAML syntax, JSON protocol, validator template, and design patterns you will need throughout this workflow
- Do NOT proceed without reading it — the reference is essential for correct hook syntax and validator architecture

### 2. Read and Understand the Target

- Read the prompt file at `TARGET_PATH`
- If `TARGET_PATH` is empty → STOP and report: `"Usage: Harden <path-to-prompt>"`
- If file not found → STOP with "File not found: {TARGET_PATH}" error
- Identify:
  - **Type**: command (`.claude/commands/`), agent (`.claude/agents/`), or skill workflow
  - **Current frontmatter fields**: model, allowed-tools/tools, existing hooks (if any)
  - **Prompt level**: what does this prompt do?

### 3. Analyze Tool Usage and Outputs

Determine what the prompt **does** and what can go wrong:

- **Which tools does it use?** (Read, Edit, Write, Bash, Task, Skill, etc.)
- **What file types does it produce/modify?** (CSV, JSON, HTML, Markdown, Python, etc.)
- **What are the correctness criteria?** Examples:
  - Structural: file must parse, required fields/sections present
  - Semantic: values must be consistent, math must check out
  - External: output must pass a linter, tests must pass
- **What has gone wrong before?** (if the user mentions past failures)

Present findings to the user:

```
Target: <path>
Type: <command|agent|skill workflow>
Tools used: <list>
Output types: <list>
Potential validation points:
- <what could be validated>
- <another validation point>
```

### 4. Recommend Hook Strategy

Based on the analysis, recommend which hook type(s) to use:

<hook-strategy-decision>

- If the prompt produces/modifies **individual files** and correctness can be checked per-file → recommend **PostToolUse** with matcher
  - Matcher targets: the tools that touch the files (e.g., `"Edit|Write"`)
  - Benefit: immediate feedback, agent self-corrects before proceeding

- If the prompt has a **final deliverable** that should be validated as a whole → recommend **Stop**
  - Benefit: checks the end result, can run cross-file validation
  - Can chain multiple validators for different concerns

- If both apply → recommend **PostToolUse + Stop** combination
  - PostToolUse for per-file correctness during execution
  - Stop for final cross-cutting validation

- If the prompt should be **prevented from certain actions** → recommend **PreToolUse** with matcher
  - Matcher targets: the tools to guard (e.g., `"Write|Edit"` for protected paths)

</hook-strategy-decision>

Present recommendation to the user and wait for confirmation before proceeding.

```
Recommended strategy:
- <PostToolUse|Stop|Both|PreToolUse>: <reason>

Validators needed:
1. <validator-name>.py — <what it validates>
2. <validator-name>.py — <what it validates> (if multiple)
```

### 5. Design Validator Logic

For each recommended validator, define:

<validator-design-loop>
- **Name**: descriptive, kebab-case (e.g., `markdown-structure-validator.py`)
- **Hook type**: PostToolUse or Stop
- **What it checks**: specific validation rules
- **Pass condition**: when to return `{}`
- **Fail condition**: when to return `{"decision": "block", "reason": "..."}`
- **Dependencies**: Python libraries needed (pandas, beautifulsoup4, jsonschema, etc.)
- **Graceful degradation**: what to do when target files don't exist yet
- **Error format**: how to make error messages actionable for the agent
</validator-design-loop>

### 6. Write Validator Scripts

For each validator, create a Python script following the standard template loaded in Step 1:

<write-validators-loop>

- Use the PEP 723 shebang pattern:
  ```python
  #!/usr/bin/env -S uv run --script
  # /// script
  # requires-python = ">=3.11"
  # dependencies = [<libraries>]
  # ///
  ```

- Include the logging pattern (LOG_FILE in same directory as script)

- For **PostToolUse** validators — extract file path from stdin JSON:
  ```python
  hook_input = json.loads(sys.stdin.read())
  file_path = hook_input.get("tool_input", {}).get("file_path")
  ```

- For **Stop** validators — resolve target path via argv or default:
  ```python
  target = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_PATH
  ```

- Apply design patterns:
  - Graceful degradation (pass when target files don't exist yet)
  - File-type filtering (skip irrelevant files silently)
  - Error count limiting (max 5 errors + "and N more")
  - Actionable error messages (include file, line, expected vs actual, fix suggestion)

- Save to `.claude/hooks/validators/<name>.py` in the target project
- Make executable: `chmod +x <script-path>`

</write-validators-loop>

### 7. Add Hooks to Target Prompt

Edit the target prompt's YAML frontmatter to include the hooks block.

- If frontmatter exists → add `hooks:` field
- If frontmatter does not exist → create it with `---` delimiters

**For commands** (`.claude/commands/`):
```yaml
hooks:
  PostToolUse:
    - matcher: "<tool-regex>"
      hooks:
        - type: command
          command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/<name>.py"
```

**For agents** (`.claude/agents/`):
```yaml
hooks:
  Stop:
    - hooks:
        - type: command
          command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/<name>.py"
```

- Always quote `$CLAUDE_PROJECT_DIR` for path safety
- Verify the hooks YAML is syntactically correct (indentation matters)

### 8. Verify Setup

Run a quick verification:

- [ ] Validator script exists at the expected path
- [ ] Validator script is executable (`chmod +x`)
- [ ] Validator script runs without import errors: `echo '{}' | uv run <script>`
- [ ] Target prompt frontmatter has correct hooks YAML syntax
- [ ] Hook matcher (if PostToolUse) targets the right tools
- [ ] Validator returns `{}` for valid input (no false blocks)

If any check fails → fix the issue before reporting success.

## Instructions

- **Always read the target prompt first** — never recommend hooks without understanding what the prompt does
- **Prefer Stop hooks** for simple cases — they're less invasive and catch end-result issues
- **Use PostToolUse sparingly** — only when per-file real-time validation is clearly needed (e.g., CSV data integrity, structured file formats)
- **Don't over-validate** — check what matters, not everything. A validator for a simple text output is overkill
- **Error messages must be actionable** — the agent needs to know what's wrong AND how to fix it. Include file name, expected value, actual value, and fix suggestion
- **Graceful degradation is mandatory** — validators must pass when target files don't exist yet (the agent hasn't created them)
- **One validator = one concern** — separate structural validation from content validation. Chain via multiple Stop hooks if needed
- **Respect existing hooks** — if the target already has hooks, add to them, don't replace

## Output

```
✅ Prompt hardened: <TARGET_PATH>

Strategy: <PostToolUse|Stop|Both>
Validators added:
- <validator-name>.py — <what it checks> (<PostToolUse|Stop>)

Hooks YAML added to frontmatter:
- <hook-type>: <matcher if applicable>

Test: echo '{}' | uv run <validator-path>
```
