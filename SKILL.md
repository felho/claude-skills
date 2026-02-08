---
name: LearnFromVideo
description: Extract insights and patterns from YouTube coding videos and their GitHub repos. USE WHEN analyze video OR learn from video OR extract lessons OR video insights OR IndyDevDan OR agentic engineer video.
---

# LearnFromVideo

Extract actionable insights, patterns, and improvement ideas from YouTube coding tutorial videos — optionally cross-referenced with the video's companion GitHub repository.

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **Analyze** | "analyze video", "learn from", "extract insights", URL provided | `Workflows/Analyze.md` |

## Tools Used

- `yt-transcript` — YouTube transcript download (installed globally)
- `yt-dlp` — Video metadata and description extraction (installed via homebrew)
- `git clone --depth 1` — Shallow clone of companion repos

## Examples

**Example 1: Full analysis with GitHub repo**
```
User: "/LearnFromVideo https://www.youtube.com/watch?v=4_2j5wgt_ds"
-> Downloads transcript via yt-transcript
-> Extracts metadata + description via yt-dlp
-> Finds GitHub repo link in description
-> Clones repo, analyzes structure and key files
-> Cross-references transcript context with code patterns
-> Generates Markdown report with insights and improvement ideas
```

**Example 2: Video without GitHub repo**
```
User: "Learn from this video: https://youtu.be/abc123"
-> Downloads transcript and metadata
-> No GitHub link found in description
-> Analyzes transcript only for patterns and concepts
-> Generates report based on verbal explanations
```

**Example 3: Focused analysis**
```
User: "Analyze this video, focus on the hook patterns: https://..."
-> Full analysis pipeline
-> Report emphasizes the specific topic requested
-> Improvement ideas filtered to the focus area
```
