# dev-portfolio Plugin Design

> Design approved: 2026-02-19

## Overview

Claude Code plugin that automatically captures development session data and generates portfolio content (TIL blog posts, project summaries, resume bullets, Claude Code experience posts) as GitHub Pages drafts via PR workflow.

## Requirements

- **Output**: Hugo-based GitHub Pages site (`YYong91.github.io`)
- **Content types**: TIL, project summary, resume bullets, Claude Code experience
- **Workflow**: Draft auto-generate → PR review → merge to publish
- **Trigger**: `/portfolio log` (raw capture) + `/portfolio generate` (draft creation)
- **Scope**: Universal Claude Code plugin (works across any project)

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Any Claude Code Session (CUBIG, personal, etc) │
└──────────────┬──────────────────────────────────┘
               │ /wrap or /portfolio log
               ▼
┌──────────────────────────────┐
│  dev-portfolio plugin        │
│  session-extractor agent     │  → Extract structured raw JSON
│           ▼                  │
│  ~/.claude/portfolio/raw/    │  → Local accumulation
└──────────────────────────────┘
               │ /portfolio generate
               ▼
┌──────────────────────────────┐
│  Generation Pipeline         │
│  (4 parallel agents)         │
│  til-writer | project-summ   │
│  resume-crafter | claude-exp │
└──────────────┬───────────────┘
               │ gh pr create
               ▼
┌──────────────────────────────┐
│  YYong91.github.io           │
│  PR → Review → Merge         │
│  → GitHub Actions → Deploy   │
└──────────────────────────────┘
```

## Plugin Structure

```
dev-portfolio/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata
├── commands/
│   ├── portfolio-log.md         # /portfolio log — raw log capture
│   ├── portfolio-generate.md    # /portfolio generate — draft creation + PR
│   ├── portfolio-setup.md       # /portfolio setup — initial configuration
│   ├── portfolio-status.md      # /portfolio status — raw log inventory
│   └── portfolio-help.md        # /portfolio help — usage guide
├── skills/
│   ├── session-logging/
│   │   └── SKILL.md             # Session data extraction methodology
│   └── draft-generation/
│       └── SKILL.md             # Content type generation guide
├── agents/
│   ├── session-extractor.md     # Session → raw JSON extraction
│   ├── til-writer.md            # TIL blog post generation
│   ├── project-summarizer.md    # Project description update
│   ├── resume-crafter.md        # Resume bullet generation
│   └── claude-exp-writer.md     # Claude Code experience writing
└── README.md
```

## Commands

### /portfolio setup

Initial configuration (run once after install):
- GitHub repo name: `YYong91.github.io`
- Local clone path: `~/projects/YYong91.github.io`
- Language preference: ko/en
- Author name

Saves to `~/.claude/portfolio/config.json`.

### /portfolio log

Captures current session data as a structured raw log:

1. Collect git context (branch, commits, diff stats)
2. Run `session-extractor` agent to analyze conversation
3. Save raw JSON to `~/.claude/portfolio/raw/YYYY-MM-DD-{project}-{ticket}.json`
4. Print summary to user

### /portfolio generate

Generates draft content from accumulated raw logs:

1. Scan `~/.claude/portfolio/raw/*.json` for unprocessed logs
2. Present options to user (which content types to generate)
3. Run selected agents in parallel
4. Preview generated content
5. Create PR to personal repo
6. Mark raw logs as processed

### /portfolio status

Shows inventory of raw logs:
- Total logs
- Unprocessed vs processed
- Content hints distribution (how many TILs, projects, etc.)

### /portfolio help

Displays usage guide with all commands and examples.

## Agents

| Agent | Model | Role |
|-------|-------|------|
| session-extractor | haiku | Extract structured raw log from session context |
| til-writer | sonnet | Transform raw logs → TIL blog posts |
| project-summarizer | sonnet | Aggregate raw logs → project description updates |
| resume-crafter | sonnet | Synthesize raw logs → resume achievement bullets |
| claude-exp-writer | sonnet | Extract Claude Code tool usage → experience posts |

## Raw Log Schema (v1.0)

```json
{
  "schema_version": "1.0",
  "date": "2026-02-19",
  "project": "core_backend",
  "branch": "feature/TD-416-billing-bugfix",
  "ticket": "TD-416",
  "session_id": "abc123",
  "processed": false,

  "summary": "Short description of what was accomplished",

  "commits": [
    {
      "hash": "17af6b4",
      "message": "fix(billing): fix organization_id remnants",
      "files_changed": 15
    }
  ],

  "topics": ["billing", "DDD", "bugfix"],

  "tools_used": {
    "skills": ["brainstorming", "systematic-debugging"],
    "plugins": ["stripe", "session-wrap"],
    "commands": ["/wrap", "/commit"],
    "agents": ["billing-domain-expert"]
  },

  "lessons": [
    "Learned insight 1",
    "Learned insight 2"
  ],

  "key_decisions": [
    "Decision made and why"
  ],

  "code_stats": {
    "files_changed": 15,
    "insertions": 200,
    "deletions": 150
  },

  "content_hints": {
    "til_worthy": true,
    "project_update": false,
    "resume_worthy": false,
    "claude_experience": true
  }
}
```

## GitHub Pages Site (Hugo)

```
YYong91.github.io/
├── hugo.toml
├── content/
│   ├── _index.md                # Main page (profile intro)
│   ├── posts/
│   │   ├── til/                 # TIL category
│   │   └── claude-code/         # Claude Code experience category
│   ├── projects/                # Project portfolio
│   │   ├── _index.md
│   │   └── cubig-billing.md
│   └── resume/
│       └── _index.md
├── data/
│   └── resume.yaml              # Structured resume data
├── .github/
│   └── workflows/
│       └── deploy.yml           # Hugo build → GitHub Pages
└── themes/
```

### Content Front Matter

```yaml
---
title: "DDD Aggregate version management"
date: 2026-02-19
categories: ["til"]
tags: ["DDD", "billing", "architecture"]
project: "CUBIG Backend"
source_sessions: ["2026-02-19-cubig-TD416"]
---
```

### PR Workflow

1. `/portfolio generate` creates branch `content/2026-02-19-batch`
2. Adds generated .md files to `content/`
3. Opens PR via `gh pr create`
4. User reviews on GitHub → edits if needed → merges
5. GitHub Actions builds Hugo → deploys to Pages

## Configuration

`~/.claude/portfolio/config.json`:

```json
{
  "repo_name": "YYong91.github.io",
  "local_path": "/Users/seungyong/projects/YYong91.github.io",
  "language": "ko",
  "author": "Seungyong",
  "raw_log_dir": "~/.claude/portfolio/raw",
  "auto_log_on_wrap": true
}
```

## Implementation Phases

### Phase 1: Foundation
- Hugo site scaffold + GitHub Actions deploy
- Plugin skeleton (plugin.json, README)
- `/portfolio setup` command
- `/portfolio help` command

### Phase 2: Raw Log Pipeline
- `session-extractor` agent
- `/portfolio log` command
- Raw log schema + local storage
- `/portfolio status` command

### Phase 3: Content Generation
- 4 writer agents (TIL, project, resume, Claude exp)
- `/portfolio generate` command
- PR creation workflow
- Draft generation skill

### Phase 4: Polish
- Theme customization
- Resume page with structured data
- Integration with session-wrap (/wrap hook)
