---
title: "dev-portfolio: Claude Code 플러그인"
description: "세션 데이터에서 블로그 포스트를 자동 생성하는 AI 에이전트 파이프라인"
date: 2026-02-19
tags: ["claude-code", "plugin", "AI agent", "automation"]
weight: 4
mermaid: true
---

## 개요

Claude Code 세션 데이터에서 자동으로 블로그 포스트를 생성하는 플러그인입니다. 개발 작업을 하면 세션 로그가 남는데, `/portfolio wrap` 커맨드 하나로 TIL, 기술 블로그 글, 프로젝트 페이지를 뽑아냅니다.

실제로 이 포트폴리오 사이트의 여러 포스트가 이 플러그인으로 생성되었습니다.

**GitHub**: [YYong91/dev-portfolio](https://github.com/YYong91/dev-portfolio) (private)

## 에이전트 파이프라인

<div class="mermaid">
graph LR
SESSION["Claude Code 세션"]
subgraph "Phase 1: 추출"
SE["session-extractor<br/>(haiku)"]
end
subgraph "Phase 2: 생성"
TW["til-writer<br/>(sonnet)"]
CW["claude-exp-writer<br/>(sonnet)"]
PS["project-summarizer<br/>(sonnet)"]
RC["resume-crafter<br/>(sonnet)"]
end
RAW["Raw Log (JSON)"]
OUT["Hugo Markdown"]
SESSION --> SE
SE --> RAW
RAW --> TW
RAW --> CW
RAW --> PS
RAW --> RC
TW --> OUT
CW --> OUT
PS --> OUT
RC --> OUT
</div>

## 구성

| 구성 요소 | 수량 | 역할 |
|-----------|------|------|
| **에이전트** | 5개 | session-extractor, til-writer, claude-exp-writer, project-summarizer, resume-crafter |
| **커맨드** | 6개 | setup, log, generate, status, help, wrap |
| **스킬** | 2개 | session-logging, draft-generation |

## 핵심 설계

### 🎯 2단계 파이프라인

1단계에서 가벼운 모델(haiku)이 세션 컨텍스트를 분석해 구조화된 JSON raw log를 생성합니다. 2단계에서 더 능력 있는 모델(sonnet)이 raw log를 읽고 목적별 마크다운 콘텐츠를 생성합니다.

이렇게 분리한 이유는 비용과 품질의 균형입니다. 세션 데이터 추출은 구조화 작업이라 가벼운 모델로 충분하고, 글쓰기는 톤과 구조가 중요해서 더 큰 모델이 필요합니다.

### ✍️ 스타일 가이드 내장

각 writer 에이전트에 실제 작성자의 블로그 글 스타일을 분석한 가이드가 내장되어 있습니다. AI가 쓴 것처럼 보이는 문체를 명시적으로 금지하고, 구체적인 스타일 규칙(입니다체, Before-After 구조, 표 활용, 섹션 이모지 등)을 정의했습니다.

**금지된 패턴:**
- "이번 포스트에서는 X를 알아보겠습니다"
- "결론적으로", "도움이 되셨길 바랍니다"
- Marketing tone 일체

**요구되는 패턴:**
- Problem → Cause → Solution 구조
- 비교는 표(table)로
- 구체적 숫자, 실제 에러 메시지, 파일명 포함

### 🔄 /wrap 통합

기존 session-wrap 스킬과 통합했습니다. `/portfolio wrap` 하나로 세션 분석 + 문서 업데이트 제안 + 블로그 포스트 생성이 동시에 진행됩니다.

## 커맨드 사용법

```bash
/portfolio setup      # 최초 설정 (config.json 생성)
/portfolio log        # 현재 세션을 raw log로 캡처
/portfolio generate   # raw log → 블로그 포스트 생성
/portfolio status     # raw log 목록 확인
/portfolio wrap       # /wrap + /portfolio log 통합 실행
```

## 디렉토리 구조

```
~/.claude/plugins/local/dev-portfolio/
├── .claude-plugin/plugin.json
├── agents/
│   ├── session-extractor.md    # 세션 → JSON (haiku)
│   ├── til-writer.md           # TIL 포스트 (sonnet)
│   ├── claude-exp-writer.md    # Claude 경험 포스트 (sonnet)
│   ├── project-summarizer.md   # 프로젝트 페이지 (sonnet)
│   └── resume-crafter.md       # 이력서 bullet (sonnet)
├── commands/
│   ├── portfolio-setup.md
│   ├── portfolio-log.md
│   ├── portfolio-generate.md
│   ├── portfolio-status.md
│   ├── portfolio-help.md
│   └── portfolio-wrap.md
└── skills/
    ├── session-logging/SKILL.md
    └── draft-generation/SKILL.md
```

## 기술적 특징

- **Claude Code Plugin Architecture**: 에이전트, 커맨드, 스킬의 3-tier 구조 활용
- **모델 최적화**: 작업 특성에 따라 haiku/sonnet 분리 배치
- **자기 참조 구조**: 이 플러그인이 자기 자신에 대한 포스트를 생성할 수 있음
- **Hugo 통합**: front matter 자동 생성, 카테고리/태그 분류, source_sessions 추적
