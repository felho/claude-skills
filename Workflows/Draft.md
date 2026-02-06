---
description: Turn exploration notes or a discussion into a one-pager PRD
argument-hint: [<exploration-notes-path>]
allowed-tools: Read, Write, Glob, AskUserQuestion
---

# Draft — Discussion Notes to One-Pager PRD

Transform exploration notes (or a live discussion) into a structured one-pager PRD.

## Variables

NOTES_PATH: $1 (path to exploration notes file — may be empty)
PROJECT_DIR: current working directory
OUTPUT_DIR: docs/ (relative to PROJECT_DIR, or user-specified)
TEMPLATE_PATH: references/Templates/OnePager.md (relative to CraftPRD skill directory)

## Instructions

- If `NOTES_PATH` is provided, use those notes as input. If not, ask the user to describe the idea (effectively combining Explore + Draft).
- Write the PRD in **English** regardless of conversation language.
- Keep it to **one page** — ruthlessly prioritize. A one-pager should be readable in 5 minutes.
- Focus on the **what** and **why**, not the **how**. Technical details belong in the Deepen phase.
- Do NOT include TypeScript interfaces, SQL schemas, or code blocks. If you're tempted to add code, note it as an open question for the Deepen phase.
- Use the OnePager template as the structure guide.

## Workflow

### 1. Gather Input

- If `NOTES_PATH` is provided → read the file.
  - If file not found → STOP with "Exploration notes not found: {NOTES_PATH}"
- If `NOTES_PATH` is empty → ask the user: "Describe the product or feature you want to write up. Or provide a path to exploration notes."
- Read the OnePager template from `TEMPLATE_PATH` for structure reference.

### 2. Identify Gaps

Review the input against the template sections. For any missing information, ask the user focused questions (max 3-4 questions total — don't interrogate).

### 3. Write the One-Pager PRD

Create the PRD at `OUTPUT_DIR/<kebab-case-name>-prd.md` following the OnePager template structure.

<drafting-guidelines>
- **Problem statement:** 2-3 sentences max. If you can't state the problem concisely, it's not clear enough yet.
- **Solution overview:** The core approach in 1 paragraph. What's the key insight?
- **Key decisions:** Bulleted list of decisions made so far with brief rationale.
- **Scope:** What's in and out for V1. Be explicit about boundaries.
- **Open questions:** Things that need resolution before or during implementation.
- **Success criteria:** How to know it's working.
</drafting-guidelines>

### 4. Review with User

Present the draft to the user. Ask if anything is missing, wrong, or should be reprioritized.

### 5. Suggest Next Step

Report the file path and suggest the Deepen workflow when they're ready to add technical detail.

## Report

```
One-pager PRD created: <file-path>

Sections:
- Problem: <one-line summary>
- Solution: <one-line summary>
- Key decisions: <count>
- Open questions: <count>

Next: Use the Deepen workflow to iterate and add technical detail.
```
