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
- All timestamps MUST be clickable YouTube deep links: `[HH:MM:SS](https://youtube.com/watch?v=VIDEO_ID&t=SECONDS)` where SECONDS = HH*3600 + MM*60 + SS
- When extracting GitHub links from descriptions, look for github.com URLs specifically
- Clone repos using `gh repo clone` to `$REPOS_DIR/`
- Do NOT attempt to run or build code from cloned repos — read-only analysis
- Focus on patterns and ideas applicable to our system, not just documenting the video
- If `FOCUS_AREA` is specified, weight the analysis toward that topic but don't ignore other insights
- If transcript exceeds 2000 lines, read in chunks and summarize key sections before full analysis

## Workflow

### 1. Validate Input

- If `VIDEO_URL` is empty -> STOP and report: `"Usage: Provide a YouTube URL"`
- Create directories: `mkdir -p $REPORT_DIR` and `mkdir -p $REPOS_DIR`
- Verify `$REPORT_DIR` exists: `ls $REPORT_DIR` — if it fails -> STOP and report: `"Error: REPORT_DIR does not exist: $REPORT_DIR"`

### 2. Extract Metadata, Description, and Transcript (parallel)

Run ALL THREE commands in parallel (single message, multiple Bash calls):
- `yt-dlp --skip-download --print "%(title)s" --print "%(channel)s" --print "%(upload_date)s" --print "%(duration_string)s" "VIDEO_URL"`
- `yt-dlp --skip-download --print "%(description)s" "VIDEO_URL"`
- `yt-transcript "VIDEO_URL" -o /tmp/{video-id}-transcript.txt`

Then:
- Extract the video ID from the URL for file naming
- If transcript download fails -> note in report, continue with metadata-only analysis
- Read the transcript file to load it into context

### 3. Parse Description for GitHub Links

- Search the description text for `github.com` URLs
- If multiple GitHub links found -> select the one most likely to be the companion repo (usually the first non-docs link, or the one matching the video topic)
- Store as `REPO_URL` variable

### 4. Clone and Explore Repository (conditional)

- If no `REPO_URL` found -> skip to step 5

<repo-analysis>
- If repo already exists at `$REPOS_DIR/{repo-name}` -> skip clone, reuse existing
- Run `gh repo clone {owner}/{repo-name} $REPOS_DIR/{repo-name} -- --depth 1`
- List the repo structure: `find $REPOS_DIR/{repo-name} -type f -not -path '*/.git/*'`
- Identify key files to read (max 10-12), prioritizing:
  - README.md, CLAUDE.md
  - Configuration files (.claude/settings.json, .claude/agents/*, .claude/commands/*)
  - Spec/plan files (specs/, plans/, docs/)
  - Source code entry points
  - Skip binary files, images, lock files
- Read key files in batches of 3-4 per turn (parallel Read calls) to reduce round-trips:
  <read-key-files-loop>
  - Read 3-4 files in parallel
  - If a file is empty or minimal (<5 lines) -> note in Raw Notes, look for alternative documentation
  - Note patterns, techniques, and architectural decisions
  - Cross-reference with transcript: does the video creator explain WHY this was done this way?
  </read-key-files-loop>
- If more interesting files remain beyond the cap, note them in Raw Notes for potential follow-up
</repo-analysis>

### 5. Analyze and Cross-Reference

Before writing the report, explicitly work through these sub-steps:

- Review transcript for key concepts, techniques, and verbal explanations
- If repo was cloned -> map transcript discussions to specific code/files
- Draft a list of the top 5-7 improvement ideas — verify each has a specific code or transcript reference
- Compare against our existing system:
  - Our skill structure (`~/.claude/skills/`) — what does the video's setup do differently?
  - Our hooks (`~/.claude/settings.json` hooks section)
  - Our CLAUDE.md conventions
  - Our workflow patterns
- Collect "transcript annotations": insights the creator shares verbally that aren't visible in the code

### 6. Generate Report

- Read the report format reference: `references/ReportFormat.md`
- Generate the Markdown report following that format
- Save to `$REPORT_DIR/{yyyy-mm-dd}-{short-slug}.md` (date = video upload date; video ID goes in frontmatter only)

### 7. Self-Check

Verify the generated report before presenting to user:
- [ ] File exists at expected `$REPORT_DIR` path
- [ ] Frontmatter has all required fields (`status`, `video_id`, `channel`, `date`)
- [ ] File naming follows `{yyyy-mm-dd}-{short-slug}.md` convention
- [ ] All timestamps are YouTube deep links (not plain `[HH:MM:SS]`)
- [ ] At least 3 improvement ideas included
- [ ] Each improvement idea has a Status field

If any check fails -> fix before reporting to user.

- Report the file path to the user

## Error Messages (Use Exactly)

- Missing URL: `"Usage: Provide a YouTube URL. Example: analyze this video https://youtube.com/watch?v=..."`
- Transcript failure: `"Warning: Could not download transcript. Continuing with metadata-only analysis."`
- Clone failure: `"Warning: Could not clone repository at {url}. Continuing with transcript-only analysis."`
- Directory error: `"Error: REPORT_DIR does not exist: {path}"`

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
