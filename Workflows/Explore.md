---
description: Explore a vague product idea through structured discussion
argument-hint: [<idea-description>]
allowed-tools: Read, Write, Glob, AskUserQuestion
---

# Explore — Idea to Discussion Notes

Guide a structured discussion to turn a vague product idea into captured decisions and open questions.

## Variables

IDEA_DESCRIPTION: $1 (may be empty — ask the user)
PROJECT_DIR: current working directory
OUTPUT_DIR: docs/ (relative to PROJECT_DIR, or user-specified)

## Instructions

- This workflow is **interactive** — ask questions, don't assume answers.
- Capture decisions as they are made, not at the end.
- Use the user's language for discussion, but write files in **English**.
- Do NOT write any code or technical specifications. This is a discussion phase.
- If the user starts diving into technical details too early, gently redirect: "Let's capture the what and why first — we'll get to the how in the Deepen phase."

## Workflow

### 1. Establish the Idea

- If `IDEA_DESCRIPTION` is empty → ask: "What product or feature idea would you like to explore?"
- Restate the idea back to the user in one sentence to confirm understanding.

### 2. Structured Exploration

Walk through these dimensions one at a time. For each, ask 1-2 focused questions and capture the answer before moving on.

<exploration-dimensions>

**Problem:**
- What specific problem does this solve?
- Who experiences this problem, and how painful is it?

**Solution sketch:**
- What is the core insight or approach?
- What does the user experience look like at a high level?

**Scope boundaries:**
- What is explicitly NOT in scope for V1?
- What is the smallest version that would be useful?

**Key risks:**
- What could make this fail?
- What are the biggest unknowns?

**Success criteria:**
- How would you know this is working?
- What would make this a success vs just "shipped"?

</exploration-dimensions>

### 3. Identify Open Questions

After the exploration, list any unresolved questions or topics that need further research.

### 4. Write Discussion Notes File

Create the output file at `OUTPUT_DIR/explore-<kebab-case-name>.md`:

```markdown
# Exploration: <Idea Name>

Date: <ISO date>

## Problem

<captured problem statement>

## Solution Sketch

<high-level approach>

## Scope

### In Scope (V1)
- <item>

### Out of Scope
- <item>

## Key Risks
- <risk>

## Success Criteria
- <criterion>

## Open Questions
- <question>

## Raw Discussion Notes
<key decisions and reasoning captured during discussion>
```

### 5. Suggest Next Step

Report the file path and suggest: "When you're ready to write this up as a one-pager PRD, run the **Draft** workflow."

## Report

```
Exploration complete: <idea name>

Notes saved: <file-path>

Dimensions covered: <N>/5
Open questions: <count>

Next: Use the Draft workflow to turn these notes into a one-pager PRD.
```
