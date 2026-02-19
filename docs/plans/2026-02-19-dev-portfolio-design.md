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

## Writing Style Guide (Anti-AI)

All writer agents MUST follow these rules to produce natural, human-sounding content.

### Style: 기술 블로그체

- 해요체 + 간결 (velog/tistory 느낌)
- 짧은 문단, 핵심만
- 코드 예시는 실제 세션에서 나온 것 사용

### Anti-AI Rules

| Banned (AI smell) | Replace with |
|---|---|
| "이번 포스트에서는 X를 알아보겠습니다" | 바로 문제 상황이나 결과부터 시작 |
| "~에 대해 살펴보겠습니다" | 실제 에러 메시지나 상황 묘사로 시작 |
| "이 글이 도움이 되셨길 바랍니다" | 자연스럽게 끝내거나 다음 할 일 언급 |
| "결론적으로 / 요약하자면" | 굳이 요약 안 해도 됨 |
| 일반론 나열 ("X란 ~입니다") | raw log의 실제 경험 기반 서술 |
| 완벽한 해결 과정만 서술 | 삽질 과정, 실패한 시도, 왜 헤맸는지도 포함 |
| 모든 단락이 비슷한 길이 | 짧은 문장과 긴 설명 자연스럽게 섞기 |
| 과도한 접속사 ("또한", "더불어", "아울러") | 그냥 끊어서 쓰기 |

### Good Example

```markdown
## organization_id를 customer_id로 바꾸는 삽질기

billing 서비스에서 `organization_id`를 `customer_id`로 rename하는 작업을 했어요.
99개 파일을 고쳐야 했는데, 단순 find-replace로 될 줄 알았거든요.

근데 아니었어요.

`LowCreditAlertData`는 외부로 나가는 포트라서 `organization_id`를 유지해야 했고,
Stripe metadata에 박힌 `organization_id`는 변경 자체가 불가능했어요.
결국 WebhookUseCase에 CustomerResolver를 끼워넣어서 변환하는 방식으로 해결했어요.
```

### Bad Example (AI smell)

```markdown
## organization_id에서 customer_id로의 마이그레이션

이번 포스트에서는 billing 서비스에서 organization_id를 customer_id로
마이그레이션하는 과정에 대해 알아보겠습니다.

마이그레이션 작업에서 가장 중요한 것은 외부 인터페이스의 호환성을 유지하는 것입니다.
또한, 외부 서비스의 데이터는 변경이 불가능하다는 점도 고려해야 합니다.

이 글이 비슷한 작업을 진행하시는 분들께 도움이 되셨길 바랍니다.
```

### Agent Prompt Principle

Writer agents receive raw log data (actual errors, actual code, actual decisions) and must:
1. Write as if they ARE the developer who experienced it
2. Include specific details from the raw log (error messages, file names, numbers)
3. Show the messy reality: what went wrong, what was tried, what finally worked
4. Use the developer's natural voice, not a tutorial voice

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
