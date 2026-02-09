# Apply Workflow Improvements

Apply approved improvements from the Analyze workflow to a workflow file.

Execute the `Workflow` section after the user has reviewed and approved proposals from the Analyze workflow.

## Variables

WORKFLOW_PATH: Path to the workflow .md file to modify
APPROVED_IMPROVEMENTS: List of improvement IDs or "all" (from user's response to Analyze report)

## Instructions

- This workflow is ONLY invoked after the Analyze workflow has presented its findings
- Only apply improvements the user explicitly approved — never apply rejected or unmentioned proposals
- Preserve existing workflow structure — edit in place, don't rewrite from scratch
- After applying changes, re-read the file to verify edits landed correctly
- If an improvement conflicts with another, flag it to the user before applying

## Workflow

### 1. Load Context

- Read the current workflow file at `WORKFLOW_PATH`
- Review the approved improvements from the Analyze report (should be in conversation context)

### 2. Plan Edits

- For each approved improvement, identify:
  - Which section of the workflow to modify
  - What specific text to add, change, or remove
  - Whether it affects step numbering (renumber if needed)
- If improvements conflict with each other -> ask user which takes priority

### 3. Apply Improvements

For each approved improvement in priority order (P0 first):
<apply-improvements-loop>
- Apply the edit using the Edit tool
- Verify the edit landed correctly (Read the modified section)
- If the edit changes step numbers -> renumber subsequent steps
</apply-improvements-loop>

### 4. Final Verification

- Read the complete modified workflow file
- Verify:
  - [ ] Step numbering is sequential and correct
  - [ ] No orphaned references to old step numbers
  - [ ] New instructions are consistent with existing ones
  - [ ] Workflow still flows logically from start to end

### 5. Present Changes

- Show a summary of all changes applied
- Ask user if they want to commit

## Report

After applying improvements:

```
Applied {count} improvements to {workflow-path}:

P0:
- [improvement description] -> [section modified]

P1:
- [improvement description] -> [section modified]

Verification: All checks passed.

Ready to commit? (Use /UseGit)
```
