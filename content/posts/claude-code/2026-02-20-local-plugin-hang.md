---
title: "Claude Code가 안 켜질 때 — 로컬 플러그인 삽질 기록"
date: 2026-02-20
categories: ["claude-code"]
tags: ["claude-code", "plugin", "debugging", "troubleshooting"]
---

`claude`를 쳤습니다. 아무것도 나오지 않았습니다.

에러도 없고, 스피너도 없습니다. VSCode 익스텐션도, 맥 데스크톱 앱도 똑같이 먹통이었습니다. 3시간 동안 Claude 없이 Claude를 고쳐야 했습니다.

## 🔍 증상

터미널에서 `claude`를 입력하면 프롬프트가 돌아올 뿐 출력이 없습니다. 로딩 중인지, 에러인지 알 수가 없습니다.

단서는 하나뿐이었습니다. 직전에 로컬 플러그인 설정을 건드렸다는 것.

## 🕵️ 원인 추적

Claude Code 없이 Claude Co-Work(웹)으로 디버깅을 시작했습니다. "플러그인 설정 건드렸다가 망가졌다"가 전부인 상황이었습니다.

시도 순서:

1. 터미널 종료 후 재시작
2. iTerm2 → 기본 터미널로 교체
3. `pkill -f claude`로 프로세스 강제 종료
4. `~/.claude/installed_plugins.json` 초기화
5. 플러그인 항목을 하나씩 제거 후 재시작

`installed_plugins.json`에서 `dev-portfolio@local` 항목을 제거했을 때 정상 부팅됐습니다.

## 💥 원인

**`installed_plugins.json`을 수동으로 편집했기 때문입니다.**

Claude Code 플러그인은 `/plugin` 명령어로 설치해야 합니다. `installed_plugins.json`을 직접 편집하는 것은 지원되지 않는 방법이며, 잘못된 경로나 형식으로 인해 무한 대기 상태에 빠질 수 있습니다.

## 🔧 해결 방법

### 1. 문제 제거

`installed_plugins.json`에서 문제가 되는 항목을 제거합니다:

```bash
# ~/.claude/plugins/installed_plugins.json 열어서
# 문제가 되는 플러그인 항목 제거
```

로컬 플러그인 관련 항목(심볼릭 링크, marketplace 설정 등)을 모두 제거하면 Claude Code가 정상 부팅됩니다.

### 2. 올바른 로컬 플러그인 설치

Claude Code **내에서** `/plugin` 명령어를 사용해야 합니다:

```bash
# 1. Marketplace 추가
/plugin marketplace add /path/to/your-plugin

# 2. 플러그인 설치
/plugin install your-plugin@marketplace-name

# 3. Claude Code 재시작
```

또는 개발/테스트 목적이라면 설치 없이 사용:

```bash
claude --plugin-dir /path/to/your-plugin
```

## ✅ 로컬 플러그인 최소 구성

플러그인에 필수로 필요한 것은 `.claude-plugin/plugin.json` 파일뿐입니다:

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # 필수! 플러그인 메타데이터
├── commands/                # 선택: 슬래시 명령어
├── skills/                  # 선택: AI 스킬
└── agents/                  # 선택: 커스텀 에이전트
```

```json
// .claude-plugin/plugin.json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "Plugin description"
}
```

**주의**: `manifest.json`이나 `.mcp.json`을 루트에 만들 필요 **없습니다**. `.claude-plugin/plugin.json`만 있으면 됩니다.

## 📚 교훈

- ❌ `installed_plugins.json` 수동 편집 금지
- ✅ `/plugin` 명령어로 설치
- ✅ 개발 중이라면 `--plugin-dir` 플래그 사용
- ✅ 공식 문서 참조: https://code.claude.com/docs/en/plugins
