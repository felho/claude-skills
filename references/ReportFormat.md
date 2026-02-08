# Report Format Reference

The Analyze workflow generates a Markdown report saved to the memory system at `LEARNINGS/LearningsFromVideos/`.

## File Naming

```
$REPORT_DIR/{yyyy-mm-dd}-{short-slug}.md
```

The date is the video's upload date in `yyyy-mm-dd` format. The video ID is stored only in the frontmatter `video_id` field, NOT in the filename.

Example: `LEARNINGS/LearningsFromVideos/2026-02-02-claude-code-task-system.md`

## Report Structure

```markdown
---
status: idea
video_id: [video-id]
channel: [channel name]
date: [upload date]
repo: [github URL or null]
---

# [Video Title]

**Channel:** [channel name]
**Date:** [upload date]
**Duration:** [duration]
**Video:** [youtube URL]
**Repo:** [github URL or "N/A"]
**Local repo:** [~/dev/learnings-from-videos/{repo-name} or "N/A"]

## Summary

[2-3 sentence summary of what the video teaches]

## Key Concepts

### 1. [Concept Name]
[Explanation with transcript context]
> "[relevant quote from transcript]" — [timestamp](https://youtube.com/watch?v=VIDEO_ID&t=SECONDS)

[If repo available: code example or file reference]

### 2. [Concept Name]
...

## Patterns & Techniques

| Pattern | Description | Where in Repo |
|---------|-------------|---------------|
| [name] | [what it does] | [file path or N/A] |

## Transcript Annotations

Notable insights from the video creator that add context beyond the code:

- **[timestamp](https://youtube.com/watch?v=VIDEO_ID&t=SECONDS)** — [insight not visible in code]
- **[timestamp](https://youtube.com/watch?v=VIDEO_ID&t=SECONDS)** — [design rationale explained verbally]

## Improvement Ideas for Our System

### Idea 1: [Title]
- **Status:** idea | partially-implemented | fully-implemented
- **What:** [specific change]
- **Why:** [what we learned that motivates this]
- **Where:** [which skill/file/workflow to change]
- **Effort:** Low / Medium / High

### Idea 2: [Title]
...

## Raw Notes

[Any additional observations, edge cases, or things to revisit later]
```

## Frontmatter

Every report MUST have YAML frontmatter with at minimum:

| Field | Required | Values |
|-------|----------|--------|
| `status` | yes | `idea`, `partially-implemented`, `fully-implemented` |
| `video_id` | yes | YouTube video ID |
| `channel` | yes | Channel name |
| `date` | yes | Video upload date (YYYYMMDD) |
| `repo` | no | GitHub repo URL, or `null` if none |

**Status meanings:**
- `idea` — Report created, insights extracted, no changes made to our system yet
- `partially-implemented` — Some improvement ideas have been applied
- `fully-implemented` — All actionable ideas have been applied or explicitly rejected

Each improvement idea within the report also has its own status field to track individual progress.

## Guidelines

- Keep the report actionable — every section should inform a decision
- Transcript Annotations capture verbal context that code alone doesn't convey
- Improvement Ideas should reference specific parts of our existing system
- Make timestamps clickable YouTube deep links: `[HH:MM:SS](https://youtube.com/watch?v=VIDEO_ID&t=SECONDS)` where SECONDS = HH*3600 + MM*60 + SS
- If no repo exists, skip repo-dependent sections gracefully
- Update the frontmatter `status` as ideas get implemented over time
