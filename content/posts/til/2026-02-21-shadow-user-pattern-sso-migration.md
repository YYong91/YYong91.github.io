---
title: "기존 테이블을 지우지 않고 SSO를 붙이는 법 — Shadow User 패턴"
date: 2026-02-21
categories: ["til"]
tags: ["auth", "sso", "migration", "database", "fastapi", "gh-cli"]
project: "podonest"
source_sessions: ["2026-02-21-podonest-multiuser-sso"]
---

podonest에 멀티유저 + SSO를 붙이는 설계를 시작했습니다. 문제는 기존 `users` 테이블에 이미 7개 도메인 모델이 `users.id(Integer)`를 FK로 참조하고 있다는 점이었습니다. User 테이블을 갈아엎는 건 선택지가 아니었습니다.

## 🏗 Shadow User 패턴

해법은 테이블을 바꾸는 게 아니라 컬럼을 하나 추가하는 것이었습니다.

```sql
-- 기존 users 테이블에 컬럼 추가만
ALTER TABLE users ADD COLUMN auth_user_id BIGINT NULL UNIQUE;
```

`auth_user_id`가 `NULL`이면 기존 로컬 패스워드 인증 경로를, 값이 있으면 podo-auth SSO 경로를 탑니다. 두 경로가 동시에 살아있는 상태에서, 첫 SSO 로그인 시 이메일로 기존 계정을 찾아 `auth_user_id`를 채우는 방식으로 점진적 마이그레이션이 가능합니다.

```python
async def get_or_create_user(db, auth_user_id: int, email: str, name: str):
    # 1. auth_user_id로 조회
    result = await db.execute(select(User).where(User.auth_user_id == auth_user_id))
    user = result.scalar_one_or_none()
    if user:
        return user

    # 2. 첫 SSO 로그인 — 이메일로 기존 계정 탐색
    result = await db.execute(select(User).where(User.email == email))
    user = result.scalar_one_or_none()
    if user:
        user.auth_user_id = auth_user_id  # 기존 계정에 SSO 연결
        await db.commit()
        return user

    # 3. 신규 생성
    user = User(auth_user_id=auth_user_id, email=email, username=name)
    db.add(user)
    await db.commit()
    return user
```

FK 제약 때문에 마이그레이션 순서가 강제됩니다. `users` 테이블을 먼저 건드리면 하위 7개 모델이 전부 깨지므로, 반드시 컬럼 추가 → SSO 매핑 구축 → 기존 패스워드 경로 제거 순으로 진행해야 합니다.

## ✔ 가족 공유 아키텍처는 멀티테넌트가 아니다

설계 과정에서 중요한 구분 하나가 생겼습니다. `GET /books`에서 `user_id` 필터를 제거하면 가족 구성원 전체가 같은 데이터를 봅니다. 쓰기(`POST /books`)에는 `user_id`를 여전히 기록해 누가 추가했는지 추적합니다.

| | 멀티테넌트 | 가족 공유 |
|---|---|---|
| 읽기 | user_id 필터 O | user_id 필터 없음 |
| 쓰기 | user_id 기록 | user_id 기록 |
| 목적 | 데이터 격리 | 데이터 공유 |

단순히 필터 하나를 빼는 것처럼 보이지만, 아키텍처 의도는 완전히 다릅니다.

## 🔧 gh repo rename의 함정

작업 중에 레포 이름을 `podo-bookshop`에서 `podo-bookshelf`로 바꿔야 했는데, `gh repo rename` 명령에 `--name` 플래그가 없었습니다.

```bash
# 이건 안 됩니다
gh repo rename --name podo-bookshelf

# 이렇게 해야 합니다
gh api PATCH repos/<owner>/podo-bookshop -X PATCH -f name=podo-bookshelf
```

`gh` CLI가 지원하지 않는 GitHub API 기능은 `gh api`로 직접 PATCH 요청을 보내면 됩니다. `-f` 플래그로 JSON body 필드를 지정합니다.
