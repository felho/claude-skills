# LearnFromVideo

A Claude Code skill that extracts actionable insights, patterns, and improvement ideas from YouTube coding tutorial videos — optionally cross-referenced with the video's companion GitHub repository.

## Why This Skill Exists

Watching coding tutorial videos is a great way to discover new patterns and tools, but the insights often evaporate after the video ends. LearnFromVideo solves this by creating structured, persistent reports that capture not just what was shown, but why it matters and how it could improve your own system. The reports are saved to a knowledge base (`~/.claude/MEMORY/LEARNINGS/LearningsFromVideos/`) for long-term reference.

## How It Works

When you provide a YouTube URL, LearnFromVideo:

1. **Downloads the transcript** using `yt-transcript`
2. **Extracts metadata** (title, channel, date, duration) using `yt-dlp`
3. **Searches the video description** for GitHub repository links
4. **Clones the companion repo** (if found) for read-only analysis
5. **Cross-references transcript with code** — maps what the creator explains verbally to specific files and patterns in the repo
6. **Generates a structured Markdown report** with key concepts, patterns, transcript annotations, and improvement ideas for your own system

## Key Features

### Transcript Annotations

Captures insights the video creator shares verbally that aren't visible in the code. These are often the most valuable parts — the "why" behind design decisions, trade-offs considered, and lessons learned.

### Clickable Timestamps

All timestamps in the report are YouTube deep links that take you directly to that moment in the video:

```markdown
> "The key insight is that hooks run before the tool executes"
> — [00:14:23](https://youtube.com/watch?v=VIDEO_ID&t=863)
```

### Improvement Ideas

Every report includes actionable ideas for improving your own system, each with:
- **Status** tracking (idea → partially-implemented → fully-implemented)
- **Effort** estimate (Low/Medium/High)
- **Where** to make the change (specific skill/file/workflow)

### GitHub Repo Analysis

When a companion repo is found, LearnFromVideo reads key files (README, config, source entry points, specs) and maps them to transcript discussions. This reveals patterns that code alone doesn't explain.

## Report Format

Reports are saved as Markdown files with YAML frontmatter:

```markdown
---
status: idea
video_id: abc123
channel: Channel Name
date: 20260215
repo: https://github.com/user/repo
---

# Video Title

## Summary
## Key Concepts
## Patterns & Techniques
## Transcript Annotations
## Improvement Ideas for Our System
## Raw Notes
```

The `status` field tracks whether the report's improvement ideas have been acted on.

## Directory Structure

```
LearnFromVideo/
├── SKILL.md                        # Main skill definition with workflow routing
├── README.md                       # This file
├── IMPROVEMENTS.md                 # Planned enhancements (WatchTogether, Enrich, frame capture)
├── Workflows/
│   └── Analyze.md                  # Main analysis workflow (7 steps)
├── references/
│   └── ReportFormat.md             # Report template and guidelines
└── Tools/
    └── .gitkeep
```

## Prerequisites

The following tools must be installed:

| Tool | Purpose | Install |
|------|---------|---------|
| `yt-transcript` | Download YouTube transcripts | `npm install -g yt-transcript` |
| `yt-dlp` | Extract video metadata and descriptions | `brew install yt-dlp` |
| `gh` | Clone GitHub repositories | `brew install gh` |

## Usage Examples

### Example 1: Full analysis with GitHub repo

```
User: "/LearnFromVideo https://www.youtube.com/watch?v=4_2j5wgt_ds"
→ Downloads transcript (1200 lines)
→ Extracts metadata: "Advanced Claude Code Hooks" by IndyDevDan
→ Finds GitHub repo in description
→ Clones repo, reads key files (CLAUDE.md, hooks/, commands/)
→ Cross-references: "At 14:23 he explains why PreToolUse hooks are better than PostToolUse for blocking"
→ Generates report with 5 key concepts, 8 patterns, 4 improvement ideas
→ Saves to ~/.claude/MEMORY/LEARNINGS/LearningsFromVideos/2026-02-02-advanced-claude-code-hooks.md
```

### Example 2: Video without GitHub repo

```
User: "Learn from this video: https://youtu.be/abc123"
→ Downloads transcript and metadata
→ No GitHub link found in description
→ Analyzes transcript only: extracts concepts, patterns from verbal explanations
→ Report omits repo-dependent sections, focuses on transcript annotations
```

### Example 3: Focused analysis

```
User: "Analyze this video, focus on the hook patterns: https://..."
→ Full analysis pipeline runs
→ Report emphasizes hook-related concepts, patterns, and improvement ideas
→ Other insights still included but with less depth
```

## Planned Enhancements

The `IMPROVEMENTS.md` file documents planned upgrades:

- **Transcript Intelligence Pass** — Novelty detection that compares video concepts against your existing system before deep-diving. Novel concepts get more coverage than known ones.
- **Frame Capture** — Extract key video frames at important timestamps using `ffmpeg` for visual context that transcripts miss.
- **WatchTogether Workflow** — Interactive mode where you watch the video and drop highlights/thoughts in real-time. Claude acknowledges each highlight and synthesizes a report afterward.
- **Enrich Workflow** — Add user highlights, expand sections, and capture frames for an existing report without regenerating it from scratch.

## Output Location

Reports are saved to:

```
~/.claude/MEMORY/LEARNINGS/LearningsFromVideos/{yyyy-mm-dd}-{short-slug}.md
```

Where:
- `{yyyy-mm-dd}` is the video's upload date
- `{short-slug}` is a descriptive slug (e.g., `claude-code-task-system`)
- The video ID is stored in the frontmatter, not the filename

Cloned repositories are stored at:

```
~/dev/learnings-from-videos/{repo-name}/
```

## Installation

This skill is part of the `~/.claude/skills/` directory and activates automatically when Claude Code detects video-analysis requests. Ensure the prerequisite tools (`yt-transcript`, `yt-dlp`, `gh`) are installed.

## When Does It Activate?

LearnFromVideo triggers on:
- "analyze video", "learn from video", "extract lessons"
- "video insights", "IndyDevDan", "agentic engineer video"
- Any message containing a YouTube URL with analysis intent
