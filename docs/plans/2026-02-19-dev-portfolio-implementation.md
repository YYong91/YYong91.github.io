# dev-portfolio Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Claude Code plugin that captures dev session data and auto-generates portfolio content for a Hugo-based GitHub Pages site.

**Architecture:** Plugin with 5 commands + 5 agents. Raw logs stored locally (`~/.claude/portfolio/raw/`), content generated on demand, published via PR → merge → GitHub Actions deploy.

**Tech Stack:** Hugo (SSG), GitHub Actions (CI/CD), Claude Code Plugin SDK (commands/agents/skills), gh CLI (PR automation)

---

## Phase 1: Hugo Site Foundation

### Task 1: Install Hugo

**Step 1: Install Hugo via Homebrew**

Run: `brew install hugo`

**Step 2: Verify installation**

Run: `hugo version`
Expected: `hugo v0.1XX.X-extended ...`

**Step 3: Commit (no commit needed — system tool install)**

---

### Task 2: Initialize Hugo Site

**Files:**
- Create: `hugo.toml`
- Create: `content/_index.md`
- Create: `content/posts/_index.md`
- Create: `content/posts/til/_index.md`
- Create: `content/posts/claude-code/_index.md`
- Create: `content/projects/_index.md`
- Create: `content/resume/_index.md`

**Working directory:** `~/projects/YYong91.github.io`

**Step 1: Initialize Hugo in existing repo**

Run: `hugo new site . --force`
(--force because the directory already has files)

**Step 2: Add PaperMod theme as git submodule**

Run: `git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod`

**Step 3: Configure hugo.toml**

```toml
baseURL = "https://yyong91.github.io/"
languageCode = "ko"
title = "Seungyong's Dev Log"
theme = "PaperMod"

[params]
  defaultTheme = "auto"
  ShowReadingTime = true
  ShowShareButtons = false
  ShowPostNavLinks = true
  ShowBreadCrumbs = true
  ShowCodeCopyButtons = true
  ShowToc = true

[params.homeInfoParams]
  Title = "Seungyong"
  Content = "Backend Developer — Python, DDD, Microservices"

[[params.socialIcons]]
  name = "github"
  url = "https://github.com/YYong91"

[menu]
  [[menu.main]]
    identifier = "posts"
    name = "Blog"
    url = "/posts/"
    weight = 10
  [[menu.main]]
    identifier = "projects"
    name = "Projects"
    url = "/projects/"
    weight = 20
  [[menu.main]]
    identifier = "resume"
    name = "Resume"
    url = "/resume/"
    weight = 30

[outputs]
  home = ["HTML", "RSS", "JSON"]
```

**Step 4: Create section index pages**

`content/_index.md`:
```markdown
---
title: "Home"
---
```

`content/posts/_index.md`:
```markdown
---
title: "Blog"
description: "개발 기록과 기술 경험을 공유합니다"
---
```

`content/posts/til/_index.md`:
```markdown
---
title: "TIL"
description: "Today I Learned"
---
```

`content/posts/claude-code/_index.md`:
```markdown
---
title: "Claude Code"
description: "AI 코딩 도구 활용 경험"
---
```

`content/projects/_index.md`:
```markdown
---
title: "Projects"
description: "프로젝트 포트폴리오"
---
```

`content/resume/_index.md`:
```markdown
---
title: "Resume"
description: "이력서"
---
```

**Step 5: Test local build**

Run: `hugo server -D`
Expected: Site serves at http://localhost:1313

**Step 6: Commit**

```bash
git add -A
git commit -m "feat: initialize Hugo site with PaperMod theme"
```

---

### Task 3: GitHub Actions Deploy Workflow

**Files:**
- Create: `.github/workflows/hugo.yml`

**Step 1: Create GitHub Actions workflow**

`.github/workflows/hugo.yml`:
```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.145.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: Asia/Seoul
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

**Step 2: Enable GitHub Pages in repo settings**

Run: `gh api repos/YYong91/YYong91.github.io/pages -X POST -f build_type=workflow 2>/dev/null || gh api repos/YYong91/YYong91.github.io/pages -X PUT -f build_type=workflow`

**Step 3: Commit and push**

```bash
git add .github/workflows/hugo.yml
git commit -m "ci: add GitHub Actions Hugo deploy workflow"
git push origin main
```

**Step 4: Verify deployment**

Run: `gh run list --limit 1`
Wait for completion, then visit https://yyong91.github.io

---

### Task 4: Add Sample Content

**Files:**
- Create: `content/posts/til/2026-02-19-first-post.md`

**Purpose:** Verify the full pipeline works (content → build → deploy)

**Step 1: Create a sample TIL post**

`content/posts/til/2026-02-19-first-post.md`:
```markdown
---
title: "사이트 오픈"
date: 2026-02-19
categories: ["til"]
tags: ["hugo", "github-pages"]
---

개발 블로그를 GitHub Pages + Hugo로 만들었어요.

PaperMod 테마를 쓰고 있고, GitHub Actions로 자동 배포됩니다.
앞으로 개발하면서 배운 것들을 여기에 기록할 예정이에요.
```

**Step 2: Local test**

Run: `hugo server -D`
Verify post appears at http://localhost:1313/posts/til/2026-02-19-first-post/

**Step 3: Commit and push**

```bash
git add content/posts/til/2026-02-19-first-post.md
git commit -m "content: add first sample post"
git push origin main
```

---

## Phase 2: Plugin Skeleton

### Task 5: Create Plugin Directory Structure

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/.claude-plugin/plugin.json`
- Create: `~/.claude/plugins/local/dev-portfolio/README.md`

**Step 1: Create plugin directory**

```bash
mkdir -p ~/.claude/plugins/local/dev-portfolio/.claude-plugin
mkdir -p ~/.claude/plugins/local/dev-portfolio/{commands,skills,agents}
mkdir -p ~/.claude/plugins/local/dev-portfolio/skills/{session-logging,draft-generation}
```

**Step 2: Create plugin manifest**

`.claude-plugin/plugin.json`:
```json
{
  "name": "dev-portfolio",
  "description": "Auto-generate portfolio content from Claude Code sessions",
  "version": "0.1.0",
  "author": {
    "name": "YYong91"
  }
}
```

**Step 3: Create README.md**

```markdown
# dev-portfolio

Claude Code plugin that captures development session data and generates portfolio content for your GitHub Pages site.

## Quick Start

1. Test locally: `claude --plugin-dir ~/.claude/plugins/local/dev-portfolio`
2. Setup: `/portfolio setup`
3. After a session: `/portfolio log`
4. Generate content: `/portfolio generate`

## Commands

| Command | Description |
|---------|-------------|
| `/portfolio setup` | Initial configuration (repo path, language, author) |
| `/portfolio log` | Capture current session as raw log |
| `/portfolio generate` | Generate draft content from raw logs → PR |
| `/portfolio status` | Show raw log inventory |
| `/portfolio help` | Show usage guide |
```

**Step 4: Test plugin loads**

Run: `claude --plugin-dir ~/.claude/plugins/local/dev-portfolio`
Verify: no errors on startup

---

### Task 6: Create /portfolio setup Command

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/commands/portfolio-setup.md`

**Step 1: Write the command**

`commands/portfolio-setup.md`:
```markdown
---
description: Initial setup for dev-portfolio plugin. Configures repo path, language, and author.
allowed-tools: Bash, Read, Write, AskUserQuestion
---

# /portfolio setup

## Execution

1. Ask the user for configuration:
   - GitHub repo name (default: infer from gh CLI)
   - Local clone path (default: ~/projects/{repo-name})
   - Language (ko/en, default: ko)
   - Author name

2. Create config directory if needed:
   ```bash
   mkdir -p ~/.claude/portfolio/raw
   ```

3. Save config to `~/.claude/portfolio/config.json`:
   ```json
   {
     "repo_name": "{repo_name}",
     "local_path": "{local_path}",
     "language": "{language}",
     "author": "{author}",
     "raw_log_dir": "~/.claude/portfolio/raw",
     "auto_log_on_wrap": true
   }
   ```

4. Verify the local repo path exists and is a git repo.

5. Print confirmation:
   ```
   Portfolio setup complete!
   - Repo: {repo_name}
   - Path: {local_path}
   - Language: {language}
   - Raw logs: ~/.claude/portfolio/raw/
   ```
```

**Step 2: Test the command**

Run in Claude Code: `/portfolio setup`

---

### Task 7: Create /portfolio help Command

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/commands/portfolio-help.md`

**Step 1: Write the command**

`commands/portfolio-help.md`:
```markdown
---
description: Show usage guide for dev-portfolio plugin
allowed-tools: Read
---

# /portfolio help

## Execution

Display the following usage guide:

---

## dev-portfolio Usage Guide

### Commands

| Command | When to use | What it does |
|---------|------------|--------------|
| `/portfolio setup` | First time only | Configure repo path, language, author |
| `/portfolio log` | After each session | Capture session data as raw JSON |
| `/portfolio generate` | When you want new content | Generate draft posts → PR |
| `/portfolio status` | Check inventory | Show pending/processed raw logs |
| `/portfolio help` | Right now | This guide |

### Typical Workflow

1. Work on your project as usual
2. Before ending the session: `/portfolio log`
3. When you have a few logs accumulated: `/portfolio generate`
4. Review the PR on GitHub → edit if needed → merge
5. Site auto-deploys via GitHub Actions

### Raw Log Location

`~/.claude/portfolio/raw/*.json`

### Config Location

`~/.claude/portfolio/config.json`

---
```

---

### Task 8: Create /portfolio status Command

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/commands/portfolio-status.md`

**Step 1: Write the command**

`commands/portfolio-status.md`:
```markdown
---
description: Show raw log inventory — pending vs processed, content hints distribution
allowed-tools: Bash, Read, Glob
---

# /portfolio status

## Execution

1. Read config from `~/.claude/portfolio/config.json`. If not found, tell user to run `/portfolio setup` first.

2. Scan `~/.claude/portfolio/raw/*.json` for all raw log files.

3. For each file, read and categorize:
   - **Unprocessed**: `processed` is false or missing
   - **Processed**: `processed` is true

4. Aggregate content_hints across unprocessed logs:
   - TIL-worthy count
   - Project update count
   - Resume-worthy count
   - Claude experience count

5. Display summary:
   ```
   Portfolio Raw Log Status
   ========================
   Total logs:      12
   Unprocessed:      5
   Processed:        7

   Unprocessed Content Hints:
   - TIL posts:          3
   - Project updates:    2
   - Resume bullets:     1
   - Claude experiences: 2

   Oldest unprocessed: 2026-02-15
   Latest unprocessed: 2026-02-19
   ```
```

---

## Phase 3: Raw Log Pipeline

### Task 9: Create session-extractor Agent

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/agents/session-extractor.md`

**Step 1: Write the agent definition**

`agents/session-extractor.md`:
```markdown
---
name: session-extractor
description: Extract structured raw log from current Claude Code session context
tools: ["Bash", "Read", "Glob", "Grep"]
model: haiku
---

# Session Extractor

You extract structured data from the current development session for portfolio content generation.

## Your Task

Analyze the current session context and produce a JSON raw log.

## Data Collection Steps

### 1. Git Context
Run these commands to gather git data:
```bash
# Current branch
git branch --show-current

# Recent commits on this branch (not on develop/main)
git log origin/develop..HEAD --oneline 2>/dev/null || git log --oneline -10

# Change stats
git diff --stat origin/develop..HEAD 2>/dev/null || git diff --stat HEAD~5..HEAD
```

### 2. Project Context
- Project name: infer from the root directory name
- Ticket number: extract from branch name (e.g., `feature/TD-416-xxx` → `TD-416`)

### 3. Session Analysis
From the conversation history, identify:
- **summary**: One-line description of what was accomplished
- **topics**: Technical topics covered (e.g., ["DDD", "billing", "testing"])
- **lessons**: Specific things learned (concrete, not generic)
- **key_decisions**: Important decisions made and WHY
- **tools_used**: Skills, plugins, commands, agents used in this session

### 4. Content Hints
Judge whether this session is worth turning into:
- **til_worthy**: Did the developer learn something transferable?
- **project_update**: Was a significant feature/milestone completed?
- **resume_worthy**: Was this achievement impressive enough for a resume?
- **claude_experience**: Were Claude Code tools used in an interesting way?

## Output Format

Return ONLY valid JSON matching this schema:
```json
{
  "schema_version": "1.0",
  "date": "YYYY-MM-DD",
  "project": "project_name",
  "branch": "full/branch/name",
  "ticket": "TD-416 or null",
  "session_id": "from conversation or generated",
  "processed": false,
  "summary": "One line",
  "commits": [{"hash": "abc1234", "message": "commit msg", "files_changed": 5}],
  "topics": ["topic1", "topic2"],
  "tools_used": {
    "skills": [],
    "plugins": [],
    "commands": [],
    "agents": []
  },
  "lessons": ["Specific lesson 1"],
  "key_decisions": ["Decision and reasoning"],
  "code_stats": {"files_changed": 10, "insertions": 100, "deletions": 50},
  "content_hints": {
    "til_worthy": true,
    "project_update": false,
    "resume_worthy": false,
    "claude_experience": false
  }
}
```
```

---

### Task 10: Create session-logging Skill

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/skills/session-logging/SKILL.md`

**Step 1: Write the skill**

`skills/session-logging/SKILL.md`:
```markdown
---
name: session-logging
description: Use when capturing session data for portfolio. Guides session-extractor agent and raw log storage. Triggered by /portfolio log command.
---

# Session Logging

## Overview

Captures the current development session as a structured raw log for later content generation.

## Process

1. Read config from `~/.claude/portfolio/config.json`
2. Launch `session-extractor` agent (Task tool, subagent_type: general-purpose)
3. Agent analyzes git context + conversation → returns JSON
4. Validate JSON has required fields
5. Save to `~/.claude/portfolio/raw/YYYY-MM-DD-{project}-{ticket}.json`
   - If ticket is null, use branch short name
   - If file already exists, append `-2`, `-3`, etc.
6. Print summary to user

## Validation Rules

Required fields: date, project, summary, topics, content_hints
Optional but recommended: commits, lessons, key_decisions, tools_used
```

---

### Task 11: Create /portfolio log Command

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/commands/portfolio-log.md`

**Step 1: Write the command**

`commands/portfolio-log.md`:
```markdown
---
description: Capture current session as a structured raw log for portfolio content generation. Use at session end or anytime during a session.
allowed-tools: Bash, Read, Write, Glob, Grep, Task, AskUserQuestion
---

# /portfolio log

## Execution

Follow the session-logging skill (skills/session-logging/SKILL.md).

### Steps

1. **Check config**: Read `~/.claude/portfolio/config.json`. If missing, tell user to run `/portfolio setup`.

2. **Collect git context**:
   ```bash
   git branch --show-current
   git log origin/develop..HEAD --oneline 2>/dev/null || git log --oneline -5
   git diff --stat origin/develop..HEAD 2>/dev/null || git diff --stat HEAD~3..HEAD
   ```

3. **Extract session data**: Launch `session-extractor` agent via Task tool.
   The agent has access to the full conversation context and will produce structured JSON.

4. **Let user review**: Show the extracted raw log summary:
   ```
   Session Log Preview
   ===================
   Date:     2026-02-19
   Project:  core_backend
   Branch:   feature/TD-416-billing-bugfix
   Summary:  organization_id 잔재 및 DI account_service 누락 버그 수정
   Topics:   billing, DDD, bugfix
   Lessons:  2 items
   Hints:    TIL ✓  Project ✗  Resume ✗  Claude ✓
   ```
   Ask: "이 내용으로 저장할까요? 수정할 부분이 있으면 알려주세요."

5. **Save raw log**: Write JSON to `~/.claude/portfolio/raw/YYYY-MM-DD-{project}-{ticket}.json`

6. **Confirm**: "Raw log 저장 완료: ~/.claude/portfolio/raw/2026-02-19-cubig-TD416.json"
```

---

## Phase 4: Content Generation Pipeline

### Task 12: Create Writer Agents

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/agents/til-writer.md`
- Create: `~/.claude/plugins/local/dev-portfolio/agents/project-summarizer.md`
- Create: `~/.claude/plugins/local/dev-portfolio/agents/resume-crafter.md`
- Create: `~/.claude/plugins/local/dev-portfolio/agents/claude-exp-writer.md`

**Step 1: Create til-writer agent**

`agents/til-writer.md`:
```markdown
---
name: til-writer
description: Generate TIL blog posts from raw session logs. Writes in natural Korean tech blog style.
tools: ["Read", "Write", "Glob"]
model: sonnet
---

# TIL Writer

You write TIL (Today I Learned) blog posts from raw development session logs.

## CRITICAL: Anti-AI Writing Rules

You are NOT an AI writing assistant. You are ghostwriting AS the developer. Write exactly as a Korean developer would write on their personal tech blog (velog/tistory style).

### BANNED phrases (instant fail):
- "이번 포스트에서는 X를 알아보겠습니다"
- "~에 대해 살펴보겠습니다"
- "이 글이 도움이 되셨길 바랍니다"
- "결론적으로" / "요약하자면"
- "~란 무엇인가?" (textbook opening)
- "또한" / "더불어" / "아울러" (excessive connectors)

### REQUIRED style:
- 해요체 (casual polite)
- Start with the problem or result, never with definitions
- Include actual error messages, file names, numbers from the raw log
- Show the messy reality: failed attempts, confusion, debugging process
- Short paragraphs (2-3 sentences max)
- Mix sentence lengths naturally
- End naturally — no forced conclusion

## Input

You receive raw log JSON files. Use the `lessons`, `key_decisions`, and `summary` fields to write.

## Output

Hugo markdown file with front matter:

```markdown
---
title: "{natural title based on the lesson}"
date: {date from raw log}
categories: ["til"]
tags: [{topics from raw log}]
project: "{project name}"
source_sessions: ["{raw log filename}"]
---

{Post content — 300-800 words}
```

## Example Output

```markdown
---
title: "외부 포트의 필드는 함부로 rename하면 안 된다"
date: 2026-02-19
categories: ["til"]
tags: ["DDD", "billing", "refactoring"]
project: "CUBIG Backend"
source_sessions: ["2026-02-19-cubig-TD416"]
---

billing 서비스에서 `organization_id`를 `customer_id`로 rename하는 작업을 했어요.
99개 파일이었는데, 단순 find-replace로 될 줄 알았거든요.

근데 아니었어요.

`LowCreditAlertData`는 이메일 알림으로 나가는 외부 포트라서
organization_id를 그대로 써야 했어요. 여기를 바꾸면 이메일 서비스 쪽에서
터지거든요.

Stripe metadata에 박힌 `organization_id`도 마찬가지였어요.
Stripe API로 이미 저장된 metadata의 key를 변경하는 건 불가능해요.
결국 WebhookUseCase에 CustomerResolver를 새로 만들어서
webhook이 들어올 때 organization_id → customer_id로 변환하는 방식으로 우회했어요.

교훈: rename 작업 전에 "이 필드가 외부로 나가는가?"를 먼저 체크하자.
내부 도메인 필드와 외부 인터페이스 필드는 생명주기가 다르다.
```
```

**Step 2: Create project-summarizer agent**

`agents/project-summarizer.md`:
```markdown
---
name: project-summarizer
description: Aggregate raw session logs into project portfolio descriptions. Updates existing project pages with new achievements.
tools: ["Read", "Write", "Glob"]
model: sonnet
---

# Project Summarizer

You maintain project portfolio pages by aggregating development session data.

## Anti-AI Rules

Same rules as til-writer. Write as the developer, not as a copywriter.
Avoid: "최첨단 기술을 활용한", "혁신적인 솔루션", "강력한 시스템"
Instead: specific numbers, actual technologies, concrete outcomes.

## Input

- Multiple raw log JSON files for the same project
- Existing project page (if any) from content/projects/

## Output

Hugo markdown for project page:

```markdown
---
title: "{Project Name}"
description: "{one-line description}"
date: {latest update date}
tags: [{all unique topics across logs}]
weight: {1-99, lower = more important}
---

## 개요

{2-3 sentences: what the project is, what problem it solves}

## 기술 스택

{Actual technologies used, extracted from logs}

## 주요 작업

{Bullet points of significant achievements, with dates}
- **2026-02-19**: organization_id → customer_id 통일 (99파일, 레이어별 커밋 분리)
- **2026-02-13**: Customer Aggregate 도입으로 PG 매핑 내부 소유
- ...

## 아키텍처 특징

{Notable architectural decisions, 2-3 bullet points}
```
```

**Step 3: Create resume-crafter agent**

`agents/resume-crafter.md`:
```markdown
---
name: resume-crafter
description: Synthesize raw session logs into resume achievement bullets. Quantified, action-oriented.
tools: ["Read", "Write", "Glob"]
model: sonnet
---

# Resume Crafter

You write resume achievement bullets from development session data.

## Output Format

Append to `data/resume.yaml`:

```yaml
achievements:
  - date: "2026-02"
    project: "CUBIG Backend"
    bullet_ko: "DDD 패턴 기반 결제 시스템 설계 및 구현 (아키텍처 리뷰 97/100점)"
    bullet_en: "Designed and implemented DDD-based billing system (97/100 architecture review)"
    tags: ["DDD", "billing", "architecture"]
    source_sessions: ["2026-02-19-cubig-TD416"]
```

## Rules

- Use X-Y-Z formula: "Accomplished X as measured by Y by doing Z"
- Include numbers when available (files changed, review scores, etc.)
- Korean bullet + English bullet both
- Only create bullets for genuinely resume-worthy sessions (content_hints.resume_worthy == true)
- Don't inflate — keep it factual
```

**Step 4: Create claude-exp-writer agent**

`agents/claude-exp-writer.md`:
```markdown
---
name: claude-exp-writer
description: Write blog posts about Claude Code tool usage experiences from session data.
tools: ["Read", "Write", "Glob"]
model: sonnet
---

# Claude Experience Writer

You write blog posts about interesting Claude Code usage patterns.

## Anti-AI Rules

Same as til-writer. Write as the developer sharing their workflow, not a product review.

Avoid: "Claude Code는 강력한 도구입니다", "AI 시대에..."
Instead: specific workflow, actual commands used, what worked and what didn't.

## Input

Raw logs where `content_hints.claude_experience == true`.
Focus on `tools_used` field — skills, plugins, commands, agents.

## Output

Hugo markdown post under `content/posts/claude-code/`:

```markdown
---
title: "{specific usage experience title}"
date: {date}
categories: ["claude-code"]
tags: ["claude-code", {specific tools}]
---

{Post about the actual experience using Claude Code tools — 300-600 words}
{What was the task, which tools were used, what was the workflow}
{Include specific commands/skills invoked}
{What worked well, what was awkward}
```
```

---

### Task 13: Create draft-generation Skill

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/skills/draft-generation/SKILL.md`

**Step 1: Write the skill**

`skills/draft-generation/SKILL.md`:
```markdown
---
name: draft-generation
description: Use when generating portfolio content from accumulated raw logs. Guides writer agents and PR creation. Triggered by /portfolio generate command.
---

# Draft Generation

## Overview

Transforms accumulated raw session logs into draft portfolio content and creates a PR.

## Process

1. Load config from `~/.claude/portfolio/config.json`
2. Scan raw logs: `~/.claude/portfolio/raw/*.json` where `processed != true`
3. Group by content_hints → determine which agents to run
4. Present options to user via AskUserQuestion
5. Run selected agents in parallel (Task tool)
6. Show generated content preview
7. Clone/update personal repo → create branch → add files → PR
8. Mark raw logs as processed

## Anti-AI Quality Gate

Before creating the PR, review ALL generated content for AI-smell:
- Check for banned phrases from the Writing Style Guide
- Verify specific details from raw logs are included
- Ensure varied sentence structure
- If any content smells like AI, rewrite that section

## PR Template

Title: "Add {N} new posts from recent sessions"
Body:
- List of new files added
- Content type breakdown
- Source session dates
```

---

### Task 14: Create /portfolio generate Command

**Files:**
- Create: `~/.claude/plugins/local/dev-portfolio/commands/portfolio-generate.md`

**Step 1: Write the command**

`commands/portfolio-generate.md`:
```markdown
---
description: Generate draft portfolio content from accumulated raw logs and create a PR to the personal repo.
allowed-tools: Bash, Read, Write, Glob, Grep, Task, AskUserQuestion
---

# /portfolio generate

## Execution

Follow the draft-generation skill (skills/draft-generation/SKILL.md).

### Steps

1. **Load config**: Read `~/.claude/portfolio/config.json`. Exit if missing.

2. **Scan raw logs**: Read all `~/.claude/portfolio/raw/*.json` files.
   Filter for `processed != true`.
   If none found: "미처리 raw log가 없어요. `/portfolio log`로 먼저 세션을 기록해주세요."

3. **Analyze and present options** (AskUserQuestion):
   Show unprocessed log count and detected content hints.
   Let user select which content types to generate (multiSelect).

4. **Run writer agents** (parallel Task calls):
   - For each selected content type, launch the appropriate agent
   - Pass relevant raw log data as context
   - Each agent returns generated markdown content

5. **Anti-AI review**: Review all generated content.
   Check for banned phrases. If found, flag and offer to regenerate.

6. **Preview**: Show generated content to user for review.
   Ask if they want to proceed or edit.

7. **Create PR**:
   ```bash
   cd {config.local_path}
   git checkout main
   git pull origin main
   git checkout -b content/$(date +%Y-%m-%d)-batch
   # Copy generated files to appropriate content/ subdirectories
   git add content/
   git commit -m "content: add {N} new posts from recent sessions"
   git push origin content/$(date +%Y-%m-%d)-batch
   gh pr create --title "Add {N} new posts" --body "..."
   ```

8. **Mark processed**: Update raw log files with `"processed": true`.

9. **Done**: Print PR URL and summary.
```

---

## Phase 5: Polish & Integration

### Task 15: Customize Theme

- Add custom CSS if needed
- Configure search functionality
- Set up analytics (optional)
- Favicon and social images

### Task 16: Resume Page with Structured Data

- Create `data/resume.yaml` with profile, experience, skills
- Create `layouts/resume/single.html` for custom resume layout
- resume-crafter agent appends to `data/resume.yaml`

### Task 17: Integration with /wrap

- Explore adding `/portfolio log` as a step in session-wrap
- Or create a hook that suggests running `/portfolio log` after `/wrap`

---

## Execution Order Summary

| Phase | Tasks | Dependencies |
|-------|-------|-------------|
| 1: Hugo Site | Tasks 1-4 | None |
| 2: Plugin Skeleton | Tasks 5-8 | Task 1 (Hugo installed) |
| 3: Raw Log Pipeline | Tasks 9-11 | Task 5 (plugin structure) |
| 4: Content Generation | Tasks 12-14 | Tasks 9-11 (raw log pipeline) |
| 5: Polish | Tasks 15-17 | Tasks 1-14 |

Phases 1 and 2 can run in parallel. Phase 3 depends on Phase 2. Phase 4 depends on Phase 3.
