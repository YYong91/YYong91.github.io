---
title: "git post-merge hook으로 홈 서버 배포 자동화하기"
date: 2026-02-22
categories: ["til"]
tags: ["git", "git-hooks", "docker", "deployment", "automation"]
project: "podonest"
source_sessions: ["2026-02-22-podo-multi-testing"]
---

PR을 main에 머지할 때마다 홈 서버에 SSH 접속해서 `git pull`, `docker compose up -d --build`를 손으로 치고 있었습니다. 머지할 때마다 반복되는 단순 작업이었습니다.

## 🔧 git post-merge hook

git에는 특정 이벤트 이후 자동으로 스크립트를 실행하는 hook 기능이 있습니다. `post-merge`는 `git merge`가 완료된 직후 실행되는데, `git pull`도 내부적으로 fetch + merge 조합이기 때문에 여기서도 동일하게 발동합니다.

즉, 서버에서 `git pull`만 하면 나머지는 hook이 알아서 처리합니다.

hook 파일은 `.git/hooks/post-merge`에 위치합니다.

```bash
#!/bin/bash
set -e
cd /home/yyong/Developer/podo-budget
docker compose up -d --build
echo "✅ podo-budget 자동 배포 완료"
```

파일을 만든 뒤 실행 권한을 줘야 합니다.

```bash
chmod +x .git/hooks/post-merge
```

이후 배포 흐름은 다음과 같이 단순해집니다.

| 이전 | 이후 |
|------|------|
| SSH 접속 → `git pull` → `docker compose up -d --build` | SSH 접속 → `git pull` |

## ⚠️ .git/hooks는 git이 추적하지 않는다

`.git/` 디렉토리 자체는 git 추적 대상에서 제외됩니다. 저장소를 새로 clone하면 hook이 사라지고 다시 수동으로 설정해야 합니다.

이를 해결하는 방법은 hook 파일을 레포에 커밋해두고 심볼릭 링크로 연결하는 것입니다.

```bash
# 레포 안에 hooks 디렉토리를 만들어 관리
mkdir -p .githooks
cp .git/hooks/post-merge .githooks/post-merge

# 심볼릭 링크로 연결
ln -sf ../../.githooks/post-merge .git/hooks/post-merge
```

또는 git 2.9 이상에서는 `core.hooksPath`를 사용할 수 있습니다.

```bash
git config core.hooksPath .githooks
```

이렇게 하면 clone 후 `git config core.hooksPath .githooks` 한 번만 실행하면 됩니다.

## 🏁 남은 과제

이번 세션에서는 배포 후 `alembic upgrade head`를 컨테이너 안에서 수동으로 실행했습니다.

```bash
docker compose exec backend alembic upgrade head
```

DB 마이그레이션은 배포 때마다 필요한 작업이므로, hook에 추가하거나 `docker-compose.yml`의 entrypoint에 포함시키는 것이 맞습니다. 수동 단계가 하나라도 남아 있으면 언젠가 빠뜨리게 됩니다.
