# Self-Validation Hooks Reference

Hooks embedded in command/agent/skill YAML frontmatter enable **specialized self-validation** — each prompt carries its own deterministic validation logic.

> **Core principle:** A focused agent with one purpose outperforms an unfocused agent with many purposes. Specialized hooks validate only what matters for that specific agent.

---

## Hook Types

| Hook Event | When It Fires | Has Matcher? | Best For |
|------------|---------------|--------------|----------|
| `PreToolUse` | Before a tool call executes | Yes | Blocking dangerous operations |
| `PostToolUse` | After a tool call completes | Yes | Validating individual file operations |
| `Stop` | When the agent/command finishes | No | Final/global validation across all outputs |

**Placement strategy:**
- **PostToolUse** — real-time validation after each Read/Edit/Write. Use when you need immediate feedback per file operation. The agent self-corrects before proceeding.
- **Stop** — end-of-run validation. Use when you need to check the overall result (all files, cross-file consistency, final output quality). Can chain multiple validators.
- **PreToolUse** — guard rails. Use to prevent the agent from taking specific actions (e.g., blocking writes to protected files).

---

## YAML Frontmatter Syntax

### PostToolUse with Matcher

```yaml
hooks:
  PostToolUse:
    - matcher: "Read|Edit|Write"
      hooks:
        - type: command
          command: "uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/my-validator.py"
```

- `matcher` is a **regex** that matches against `tool_name` (e.g., `Read`, `Edit`, `Write`, `Bash`)
- Pipe-separated for multiple tools: `"Read|Edit|Write"`
- Only fires when the matched tool is used

### Stop with Single Validator

```yaml
hooks:
  Stop:
    - hooks:
        - type: command
          command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/my-validator.py"
```

- No `matcher` field — Stop hooks always fire when the agent/command finishes
- Quote `$CLAUDE_PROJECT_DIR` for path-safety: `\"$CLAUDE_PROJECT_DIR\"`

### Stop with Multiple Validators

```yaml
hooks:
  Stop:
    - hooks:
        - type: command
          command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/structure-validator.py"
        - type: command
          command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/content-validator.py"
```

- All listed validators run. If **any one** blocks, the agent is blocked.
- No documented priority/ordering mechanism — validators run in listed order.

### Combined PostToolUse + Stop

```yaml
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/per-file-validator.py"
  Stop:
    - hooks:
        - type: command
          command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/final-validator.py"
```

### Structure Breakdown

```
hooks:
  <HookEvent>:              # PreToolUse | PostToolUse | Stop
    - matcher: "<regex>"    # ONLY for PreToolUse/PostToolUse
      hooks:                # Array of hook entries
        - type: command     # Always "command"
          command: "..."    # Shell command to execute
```

---

## Validator Communication Protocol

Validators have **two options** for communicating results back to Claude Code. Choose one per validator.

### Option A: Stdout JSON Protocol (Recommended)

The validator always exits with code 0 and prints a JSON object to stdout. Claude Code reads the JSON to decide whether to block or allow.

**Allow** (pass validation):
```python
print(json.dumps({}))
sys.exit(0)
```

**Block** (fail validation):
```python
print(json.dumps({
    "decision": "block",
    "reason": "Human-readable error description for the agent to fix"
}))
sys.exit(0)
```

Note: exit code is 0 in both cases — the script itself succeeded. The `decision` field in the JSON controls blocking.

**Why recommended:**
- You have full control over the error message via the `reason` field
- Clean separation between "script crashed" (non-zero exit) and "validation found issues" (exit 0 + JSON block)
- The agent receives a well-formatted, actionable message you designed

### Option B: Exit Code Protocol

The validator uses its exit code to signal the result. The error message comes from stderr.

| Exit Code | Behavior |
|-----------|----------|
| 0 | Allow — validation passed, continue normally |
| 2 | Block — stderr content is fed back to Claude as the error message |
| Other | Non-blocking error — logged but does not block the agent |

```python
# Allow:
sys.exit(0)

# Block:
print("CSV has 3 missing columns: date, amount, category", file=sys.stderr)
sys.exit(2)
```

**Trade-off:** Simpler (no JSON needed), but the error message is whatever lands in stderr — which may include tracebacks, log noise, or unstructured output.

### Self-Correction Loop (Both Options)

Regardless of which option you use, blocking creates a **closed-loop self-correction cycle**:
1. Agent performs an action (writes a file, finishes work)
2. Hook fires, validator detects an issue, blocks with an error message
3. Claude Code feeds the error message back to the agent
4. Agent reads the error, attempts to fix the issue, retries
5. Hook fires again — repeat until validation passes

---

## Validator Architecture

### PEP 723 Script Pattern (uv run --script)

Declare dependencies inline in the script itself — no `requirements.txt`, `pyproject.toml`, or virtual environment needed. `uv` reads the metadata and installs dependencies into an ephemeral environment at runtime.

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.11"
# dependencies = ["pandas"]
# ///
```

- `uv run --script` reads dependencies from the script itself
- Installs in an ephemeral environment — no requirements.txt or venv needed
- Use `dependencies = []` for stdlib-only validators

### Standard Validator Template

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.11"
# dependencies = []
# ///

import json
import sys
from datetime import datetime
from pathlib import Path

LOG_FILE = Path(__file__).parent / "my-validator.log"

def log(message: str):
    """Append timestamped message to log file."""
    timestamp = datetime.now().strftime("%H:%M:%S")
    with open(LOG_FILE, "a") as f:
        f.write(f"[{timestamp}] {message}\n")

def validate():
    """Main validation logic. Returns list of error strings."""
    errors = []
    # ... domain-specific validation ...
    return errors

def main():
    stdin_data = sys.stdin.read()
    log(f"Input: {stdin_data[:200]}")

    try:
        hook_input = json.loads(stdin_data) if stdin_data.strip() else {}
    except json.JSONDecodeError:
        hook_input = {}

    errors = validate()

    if errors:
        result = {
            "decision": "block",
            "reason": "Validation failed:\n" + "\n".join(errors)
        }
    else:
        result = {}

    log(f"Result: {json.dumps(result)[:200]}")
    print(json.dumps(result))

if __name__ == "__main__":
    main()
```

### PostToolUse Input Schema

PostToolUse validators receive stdin JSON with tool context:

```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/absolute/path/to/file.csv",
    "content": "..."
  }
}
```

Extract file path:
```python
hook_input = json.loads(stdin_data)
tool_input = hook_input.get("tool_input", {})
file_path = tool_input.get("file_path")
```

### Stop Hook Input

Stop hooks fire when the agent/command **finishes**, not after a specific tool call. There is no "current file" at that point, so the stdin JSON does **not** contain a useful `tool_input` with file paths (unlike PostToolUse).

The validator needs another way to know **what to validate**. Three common strategies:

**1. CLI argument from the hook command string:**
You can hardcode a path or pass `$CLAUDE_PROJECT_DIR` directly in the YAML:
```yaml
command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/csv-validator.py apps/data/output/"
```
The validator reads it as `sys.argv[1]`:
```python
target_dir = Path(sys.argv[1]) if len(sys.argv) > 1 else DEFAULT_DIR
```

**2. Hardcoded default path in the script:**
The validator knows where the outputs live (e.g., from a project convention):
```python
DEFAULT_DIR = Path("apps/agentic-finance-review/data/mock_dataset_2026/")
```

**3. Glob discovery:**
The validator scans for files matching a pattern, useful when output locations vary:
```python
csv_files = list(Path(".").glob("**/*.csv"))
```

In practice, most Stop validators combine strategy 1 + 2: accept an optional CLI argument, fall back to a default.

### Logging Pattern

Every validator writes to its own `.log` file in the same directory:

```python
LOG_FILE = Path(__file__).parent / "my-validator.log"
```

- Timestamped entries: `[HH:MM:SS] message`
- Logs input, errors found, and result
- Provides observability — proof that validation ran and what it found

---

## Design Patterns

### 1. Graceful Degradation

Validators should **pass** when prerequisites don't exist yet:

```python
# If no target files exist yet, don't block (agent hasn't created them)
csv_files = list(target_dir.glob("**/*.csv"))
if not csv_files:
    log("No CSV files found yet, passing")
    print(json.dumps({}))
    return
```

This is critical because hooks fire on every relevant event — including early in the agent's execution before outputs exist.

### 2. Error Count Limiting

Prevent overwhelming the agent with too many errors:

```python
MAX_ERRORS = 5
if len(errors) > MAX_ERRORS:
    truncated = errors[:MAX_ERRORS]
    truncated.append(f"... and {len(errors) - MAX_ERRORS} more errors")
    errors = truncated
```

### 3. Actionable Error Messages

Include enough context for the agent to self-correct:

```python
# Bad: vague
"Balance mismatch in file"

# Good: specific with fix suggestion
f"{file.name} row {row}: Balance mismatch! "
f"Expected: ${expected:,.2f} = ${prev:,.2f} - ${withdrawal:,.2f} + ${deposit:,.2f}. "
f"Actual: ${actual:,.2f}. "
f"Fix: Set balance to ${expected:,.2f}"
```

### 4. File-Type Filtering

PostToolUse validators should skip irrelevant files silently:

```python
file_path = tool_input.get("file_path", "")
if not file_path.endswith(".csv"):
    print(json.dumps({}))  # Not our concern, pass
    return
```

### 5. Subprocess Delegation

For validators that wrap existing tools (linters, formatters):

```python
try:
    result = subprocess.run(
        ["uvx", "ruff", "check", "."],
        capture_output=True, text=True, timeout=120
    )
    if result.returncode != 0:
        errors.append(result.stdout[:500])
except FileNotFoundError:
    log("ruff not found, passing")  # Graceful degradation
```

### 6. Double Validation (Agent + Command)

When an agent delegates to a command via `Skill(prompt: '/my-command')`:
- The **command's** hooks fire at its own Stop
- Then the **agent's** hooks fire at the agent's Stop
- This provides two layers of validation

### 7. Agent-to-Command Mirroring

Each agent mirrors a corresponding command:

| Agent | Command | Shared Hook |
|-------|---------|-------------|
| `agents/normalize-csv-agent.md` | `commands/normalize-csv.md` | csv-validator.py |

The agent adds context isolation and can be spawned in parallel. The hooks are duplicated on both.

---

## Command vs Agent Frontmatter

| Field | Command (`.claude/commands/`) | Agent (`.claude/agents/`) |
|-------|-------------------------------|---------------------------|
| `name` | Not used (filename = name) | Required |
| `description` | Optional | Required (used for matching) |
| Tool restriction | `allowed-tools:` | `tools:` |
| `color` | Not available | Available (e.g., `cyan`) |
| `model` | Optional | Optional |
| `hooks` | **Identical syntax** | **Identical syntax** |
| `disable-model-invocation` | Available | Available |
| Arguments | `$1`, `$2`, `$ARGUMENTS` | Inferred from prompt |

---

## Hook Placement Decision Guide

| Scenario | Hook Type | Matcher | Why |
|----------|-----------|---------|-----|
| Validate each file after edit | PostToolUse | `"Edit\|Write"` | Immediate feedback per operation |
| Validate each file after any access | PostToolUse | `"Read\|Edit\|Write"` | Catch issues even on read |
| Check final output quality | Stop | (none) | Validates end result |
| Run linter after all code written | Stop | (none) | Cross-file analysis |
| Check structure + content separately | Stop (multiple) | (none) | Chain validators |
| Prevent writes to protected files | PreToolUse | `"Write\|Edit"` | Block before damage |
| Validate CSV per-file + final totals | PostToolUse + Stop | Both | Real-time + final |

---

## Environment Variables

| Variable | Description | Usage |
|----------|-------------|-------|
| `$CLAUDE_PROJECT_DIR` | Project root directory, set by Claude Code | Used in hook `command:` fields for absolute paths |

**Path safety:** Always quote the variable in hook commands:
```yaml
command: "uv run \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validators/my-validator.py"
```

---

## Common Validator Libraries

| Library | Use Case | PEP 723 |
|---------|----------|---------|
| `pandas` | CSV/data validation | `dependencies = ["pandas"]` |
| `beautifulsoup4` + `lxml` | HTML structure validation | `dependencies = ["beautifulsoup4", "lxml"]` |
| `jsonschema` | JSON schema validation | `dependencies = ["jsonschema"]` |
| `pyyaml` | YAML/frontmatter validation | `dependencies = ["pyyaml"]` |
| (stdlib) `json`, `csv`, `re` | Basic format checks | `dependencies = []` |
| (subprocess) `uvx ruff`, `uvx ty` | Lint/type-check delegation | `dependencies = []` |
