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

`~/.claude/plugins/local/dev-portfolio/` 폴더가 `installed_plugins.json`에 등록되어 있었지만, 폴더 안에 아무 파일도 없었습니다.

Claude Code는 시작할 때 등록된 플러그인을 순서대로 로드합니다. `manifest.json`이 없으면 로드 과정에서 무한 대기 상태에 빠집니다. 에러 메시지도 없이.

## 🔧 해결 과정

플러그인 폴더를 통째로 수술했습니다.

```bash
# 1. 기존 plugins 폴더 백업
mv ~/.claude/plugins ~/.claude/plugins.bak

# 2. 핵심 디렉토리만 복원
cp -r ~/.claude/plugins.bak/cache ~/.claude/plugins/
cp -r ~/.claude/plugins.bak/marketplaces ~/.claude/plugins/

# 3. installed_plugins.json 복원
#    plugins.bak의 installed_plugins.json에서 dev-portfolio@local 항목 제거 후 복사

# 4. local/dev-portfolio 폴더는 복원하지 않음
```

마켓플레이스에서 설치한 플러그인들은 cache에 있어서 재설치 없이 살아났습니다. `dev-portfolio` 로컬 플러그인만 새로 제대로 만들었습니다.

## ✅ 로컬 플러그인 만들 때 최소 구성

`installed_plugins.json`에 등록하기 전에 이 파일들이 먼저 있어야 합니다.

```json
// manifest.json
{
  "name": "plugin-name",
  "version": "0.1.0",
  "description": "설명"
}
```

```json
// .mcp.json (MCP 서버 없어도 빈 파일로)
{
  "mcpServers": {}
}
```

| 파일 | 없을 때 증상 |
|------|------------|
| `manifest.json` | Claude Code 시작 시 무한 대기 (무증상) |
| `.mcp.json` | MCP 서버 사용 시 로드 실패 |

플러그인 코드보다 구성 파일 먼저입니다.
