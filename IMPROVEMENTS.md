# Plan: Improve LearnFromVideo Skill

## Context

During a "watch together" session analyzing an IndyDevDan video, we identified that the current Analyze workflow has a **generalization trap** — it abstracts novel patterns into generic summaries. The root cause: the repo exploration is not transcript-driven, there's no novelty detection, and no interactive mode exists for user input during analysis. Additionally, we confirmed that `yt-dlp + ffmpeg` can extract video frames at specific timestamps in ~4 seconds — enabling visual context that the current text-only pipeline lacks.

## Changes Summary

| # | File | Action | Description |
|---|------|--------|-------------|
| 1 | `references/FrameCapture.md` | Create | Frame capture tool reference (command, storage, format) |
| 2 | `references/ReportFormat.md` | Modify | Add Visual Context section, User Highlights section, new frontmatter fields |
| 3 | `Workflows/Analyze.md` | Rewrite | Transcript-first approach, novelty detection, auto frame capture |
| 4 | `Workflows/WatchTogether.md` | Create | Interactive watch session: prepare → collect highlights → synthesize |
| 5 | `Workflows/Enrich.md` | Create | Enrich existing report with user highlights and expanded analysis |
| 6 | `SKILL.md` | Modify | Add new workflows to routing table, update triggers and tools |

All paths relative to `/Users/felho/.claude/skills/LearnFromVideo/`.

## Step 1: Create `references/FrameCapture.md`

New reference file documenting the frame capture tool.

**Contents:**
- Command: `ffmpeg -ss SECONDS -i "$(yt-dlp -g -f 'bestvideo[height<=720]' 'URL' 2>/dev/null)" -frames:v 1 -update 1 -q:v 2 OUTPUT.jpg -y 2>/dev/null`
- Storage: `$REPORT_DIR/frames/{video-id}/{video-id}-{seconds}s.jpg`
- Markdown reference: `![Frame at HH:MM:SS — description](frames/{video-id}/{video-id}-{seconds}s.jpg)`
- Performance: ~4 seconds per frame, no full video download
- Error handling: if capture fails, note in report but continue — frames are supplementary

## Step 2: Update `references/ReportFormat.md`

Extend existing format with:

**New optional frontmatter fields:**
```yaml
frames: [698, 1068]  # captured frame timestamps in seconds
highlights: 5         # user highlight count (WatchTogether/Enrich only)
```

**New section: Visual Context** (after Transcript Annotations, before Improvement Ideas):
```markdown
## Visual Context
### [HH:MM:SS](deep-link) — [description]
![Frame at HH:MM:SS — description](frames/{video-id}/{video-id}-{seconds}s.jpg)
[What this frame reveals that the transcript doesn't convey]
```

**New section: User Highlights** (after Summary, before Key Concepts — WatchTogether/Enrich only):
```markdown
## User Highlights
### Highlight 1: [paraphrased observation]
- **User said:** "[direct quote]"
- **Transcript context:** [relevant passage + timestamp]
- **Significance:** [why this matters]
```

User Highlights appears BEFORE Key Concepts to signal these are high-priority, user-validated insights.

## Step 3: Rewrite `Workflows/Analyze.md`

Current 7-step → new 8-step pipeline. Key change: **Transcript Intelligence Pass** inserted before repo exploration.

**New pipeline:**

1. **Validate Input** — same as now + `mkdir -p $REPORT_DIR/frames/{video-id}`
2. **Extract Metadata, Description, Transcript** — same as now (parallel)
3. **Transcript Intelligence Pass** (NEW) — three sub-analyses:
   - **3a Novelty Detection:** Scan transcript concepts against our system (`ls ~/.claude/skills/`, recent reports headers). Score each: `known` | `variant` | `novel`
   - **3b File/Tool Map:** Extract every file path, tool name, command mentioned in transcript with timestamps
   - **3c Frame Candidates:** Identify timestamps where visual context adds value (speaker says "let me show you", describes screen layout, shows a result). Select top 3-5
4. **Parse Description for GitHub Links** — same as now
5. **Transcript-Driven Repo Exploration** (REPLACES generic "read 10-12 key files"):
   - Use File/Tool Map from 3b to find specific repo files
   - Prioritize files related to `novel` concepts from 3a
   - Cross-reference: does code confirm/extend what speaker said?
   - Cap at 15 files, prioritized by novelty
6. **Capture Key Frames** (NEW):
   - Read `references/FrameCapture.md`
   - Capture frames at timestamps from 3c (max 5)
   - Read each frame with Read tool for multimodal analysis
   - Note visual info not present in transcript
7. **Analyze, Cross-Reference, Generate Report** (merges current steps 5+6):
   - Lead with `novel` concepts (deepest coverage)
   - Then `variant` concepts (emphasize difference from our approach)
   - Then `known` concepts (brief mention)
   - Include Visual Context section with captured frames
   - Follow ReportFormat.md
8. **Self-Check** — same as now + verify frames exist + verify novel concepts have more depth

## Step 4: Create `Workflows/WatchTogether.md`

Interactive workflow with 3 phases.

**Phase 1: Prepare** (before user starts watching)
- Download transcript + metadata + description (parallel)
- Quick pre-analysis: identify 5-7 interesting segments with timestamps
- Clone GitHub repo in background if found
- Present to user: video info, interesting segments, instructions

**Phase 2: Collect** (interactive loop while user watches)
- User drops messages with thoughts/highlights
- Claude responds with **brief ack** (1-2 sentences): confirms highlight, links to transcript timestamp
- If user describes something visual → suggest frame capture
- If user says "capture frame at HH:MM:SS" → capture immediately, confirm
- Stores each highlight: `{user_message, timestamp, transcript_context}`
- Exits when user says "done watching" or similar

**Phase 3: Synthesize** (after watching)
- Targeted repo exploration driven by user highlights (not generic)
- Novelty detection weighted by highlights (user-highlighted = deep coverage regardless)
- Read captured frames for visual analysis
- Generate report with User Highlights section + Visual Context section
- Key Concepts ordered: user-highlighted-novel > user-highlighted > novel > variant > known
- Self-check + report file path

## Step 5: Create `Workflows/Enrich.md`

Workflow to enrich an existing report.

**Steps:**
1. **Load Report** — read existing report, extract video URL from frontmatter, note current sections
2. **Collect Input** (interactive loop):
   - "expand on [topic]" → grep transcript + repo for that topic, present findings, ask to add
   - "capture frame at HH:MM:SS" → capture, describe, ask to add
   - Free-form insights → store as highlights, search transcript for context
   - "done" → exit loop
3. **Apply Enrichments** — EDIT existing report (not rewrite): add highlights, expand sections, add frames
4. **Self-Check** — verify report integrity, new frames exist, no sections lost
5. **Report** — summary of changes made

Key principle: Enrich uses targeted transcript search (Grep for keywords), NOT full re-read (unless transcript <500 lines).

## Step 6: Update `SKILL.md`

**Updated routing table:**
```markdown
| Workflow | Trigger | File |
|----------|---------|------|
| **Analyze** | "analyze video", "learn from", "extract insights", URL provided | `Workflows/Analyze.md` |
| **WatchTogether** | "watch together", "watch with me", "let's watch" | `Workflows/WatchTogether.md` |
| **Enrich** | "enrich report", "expand report", "add to report" | `Workflows/Enrich.md` |
```

**Updated Tools Used:** add `ffmpeg` for frame capture

**Add examples** for WatchTogether and Enrich workflows.

## Implementation Order

1 → 2 → 3 → 4 → 5 → 6 (references first, then workflows, then SKILL.md last)

Each step committed separately.

## Verification

After all steps:
1. Run `/LearnFromVideo analyze https://www.youtube.com/watch?v=RpUTF_U4kiw` on a fresh video to test the improved Analyze (check: novelty detection, transcript-driven exploration, auto frame capture)
2. Run `/LearnFromVideo watch together <url>` and simulate a session to test WatchTogether
3. Run `/LearnFromVideo enrich <existing-report-path>` on an existing report to test Enrich
4. Verify frames are saved correctly in `$REPORT_DIR/frames/{video-id}/`
