# Analyze Video

Extract insights, patterns, and improvement ideas from a YouTube coding tutorial video, optionally cross-referenced with its companion GitHub repository.

Execute the `Workflow` section and generate a report following the `Report` format.

## Variables

VIDEO_URL: $ARGUMENTS
REPORT_DIR: /Users/felho/.claude/MEMORY/LEARNINGS/LearningsFromVideos
REPOS_DIR: /Users/felho/dev/learnings-from-videos
FOCUS_AREA: (optional, extracted from user message if present)

## Instructions

- If `VIDEO_URL` is empty -> STOP and report: `"Usage: Provide a YouTube URL. Example: analyze this video https://youtube.com/watch?v=..."`
- All report content MUST be in English regardless of conversation language
- Transcript quotes must include timestamps for verifiability
- When extracting GitHub links from descriptions, look for github.com URLs specifically
- Clone repos using `gh repo clone` to `$REPOS_DIR/`
- Do NOT attempt to run or build code from cloned repos — read-only analysis
- Focus on patterns and ideas applicable to our system, not just documenting the video
- If `FOCUS_AREA` is specified, weight the analysis toward that topic but don't ignore other insights

## Workflow

### 1. Validate Input

- If `VIDEO_URL` is empty -> STOP and report: `"Usage: Provide a YouTube URL"`
- Create directories: `mkdir -p $REPORT_DIR` and `mkdir -p $REPOS_DIR`

### 2. Extract Video Metadata

- Run `yt-dlp --skip-download --print "%(title)s" --print "%(channel)s" --print "%(upload_date)s" --print "%(duration_string)s" "VIDEO_URL"` to get title, channel, date, duration
- Run `yt-dlp --skip-download --print "%(description)s" "VIDEO_URL"` to get the full description
- Extract the video ID from the URL for file naming

### 3. Download Transcript

- Run `yt-transcript "VIDEO_URL" -o /tmp/{video-id}-transcript.txt`
- If transcript download fails -> note in report, continue with metadata-only analysis
- Read the transcript file to load it into context

### 4. Parse Description for GitHub Links

- Search the description text for `github.com` URLs
- If multiple GitHub links found -> select the one most likely to be the companion repo (usually the first non-docs link, or the one matching the video topic)
- Store as `REPO_URL` variable

### 5. Clone and Explore Repository (conditional)

- If no `REPO_URL` found -> skip to step 6

<repo-analysis>
- If repo already exists at `$REPOS_DIR/{repo-name}` -> skip clone, reuse existing
- Run `gh repo clone {owner}/{repo-name} $REPOS_DIR/{repo-name} -- --depth 1`
- List the repo structure: `find $REPOS_DIR/{repo-name} -type f -not -path '*/.git/*'`
- Identify key files to read, prioritizing:
  - README.md, CLAUDE.md
  - Configuration files (.claude/settings.json, .claude/agents/*, .claude/commands/*)
  - Spec/plan files (specs/, plans/, docs/)
  - Source code entry points
- For each key file identified:
  <read-key-files-loop>
  - Read the file
  - Note patterns, techniques, and architectural decisions
  - Cross-reference with transcript: does the video creator explain WHY this was done this way?
  </read-key-files-loop>
</repo-analysis>

### 6. Analyze and Cross-Reference

- Review transcript for key concepts, techniques, and verbal explanations
- If repo was cloned -> map transcript discussions to specific code/files
- Identify patterns that could improve our existing system:
  - Compare against our skill structure (`~/.claude/skills/`)
  - Compare against our hooks (`~/.claude/settings.json` hooks section)
  - Compare against our CLAUDE.md conventions
  - Compare against our workflow patterns
- Collect "transcript annotations": insights the creator shares verbally that aren't visible in the code

### 7. Generate Report

- Read the report format reference: `references/ReportFormat.md`
- Generate the Markdown report following that format
- Save to `$REPORT_DIR/{yyyy-mm-dd}-{short-slug}.md` (date = video upload date; video ID goes in frontmatter only)
- Report the file path to the user

## Error Messages (Use Exactly)

- Missing URL: `"Usage: Provide a YouTube URL. Example: analyze this video https://youtube.com/watch?v=..."`
- Transcript failure: `"Warning: Could not download transcript. Continuing with metadata-only analysis."`
- Clone failure: `"Warning: Could not clone repository at {url}. Continuing with transcript-only analysis."`

## Report

After completing the analysis, output:

```
Report saved: {report-path}

Video: {title} by {channel} ({duration})
Repo: {repo-url or "none found"}
Transcript: {line-count} lines loaded

Key findings:
- {count} patterns/techniques identified
- {count} improvement ideas for our system
- {count} transcript annotations captured

Open the report to review: {report-path}
```
