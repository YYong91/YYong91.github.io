---
title: "podonest: 가족 멀티서비스 SSO 아키텍처"
description: "podo-auth를 허브로 3개 서비스가 JWT 하나로 연결되는 홈서버 기반 가족 앱 플랫폼"
date: 2026-02-21
tags: ["FastAPI", "React", "SSO", "JWT", "OAuth", "PostgreSQL", "SQLite", "Docker", "System Architecture", "Multi-user Architecture"]
weight: 3
mermaid: true
---

## 개요

홈서버에서 운영하는 세 개의 가족용 앱(독서 기록, 가계부, 인증 허브)을 단일 계정으로 연결하는 플랫폼입니다. 각 서비스가 독립적으로 개발되어 있고 DB 스키마도 다르기 때문에, 기존 데이터를 깨지 않고 SSO를 붙이는 것이 핵심 과제였습니다.

## 서비스 구성

<div class="mermaid">
graph TD
    AUTH["podo-auth<br/>auth.podonest.com<br/>FastAPI + SQLite<br/>JWT 발급 (HS256, TSID)"]

    SHELF["podo-bookshelf<br/>bookshelf.podonest.com<br/>FastAPI + SQLite<br/>React 19"]

    BUDGET["podo-budget<br/>budget.podonest.com<br/>FastAPI + PostgreSQL<br/>React 19"]

    AUTH -->|"JWT (sub=BigInt TSID)"| SHELF
    AUTH -->|"JWT (sub=BigInt TSID)"| BUDGET

    subgraph "인프라"
        DC["Docker Compose"]
        CF["Cloudflare Tunnel"]
    end

    DC --> AUTH
    DC --> SHELF
    DC --> BUDGET
    CF --> AUTH
    CF --> SHELF
    CF --> BUDGET
</div>

| 서비스 | 도메인 | DB | 인증 방식 |
|--------|--------|----|-----------|
| podo-auth | auth.podonest.com | SQLite | 자체 (JWT 발급 허브) |
| podo-bookshelf | bookshelf.podonest.com | SQLite | SSO 적용 완료 |
| podo-budget | budget.podonest.com | PostgreSQL | 자체 → SSO 전환 중 |

## 기술 스택

- **백엔드**: FastAPI (Python), SQLAlchemy ORM
- **프론트엔드**: React 19
- **인증**: JWT HS256, TSID (BigInt PK), issuer claim 기반 검증
- **DB**: SQLite (auth, bookshelf), PostgreSQL (budget)
- **인프라**: 홈서버, Docker Compose, Cloudflare Tunnel

## 주요 작업

- **2026-02-21**: 3개 서비스 인증 흐름 전체 분석 — auth_user_id(BigInt) vs budget.users.id(Integer) PK 불일치 문제 확인, 7개 도메인 모델의 FK 의존성 파악
- **2026-02-21**: Shadow User 패턴 설계 — podo-budget SSO 전환 방안 확정
- **2026-02-21**: 가족 공유 모델 정의 — podo-bookshelf 읽기/쓰기 권한 구조 재설계

## 아키텍처 특징

### Shadow User 패턴 (podo-budget SSO)

podo-budget의 `users` 테이블은 Integer PK를 사용하고, 7개의 도메인 모델(지출, 수입, 카테고리 등)이 FK로 참조하고 있습니다. podo-auth의 JWT `sub`은 BigInt TSID입니다. 두 타입이 달라 직접 교체가 불가능한 상황이었습니다.

기존 Integer PK는 그대로 두고, `auth_user_id BigInteger` 컬럼을 추가하는 방식을 택했습니다. JWT가 들어오면 email 매칭으로 Shadow User를 찾거나 최초 1회 생성합니다. 7개 도메인 모델의 마이그레이션 없이 SSO를 붙일 수 있습니다.

```
users 테이블 (변경 전)         users 테이블 (변경 후)
─────────────────────          ──────────────────────────────────
id INTEGER PK                  id INTEGER PK          ← 유지
email VARCHAR                  email VARCHAR
password_hash VARCHAR          password_hash VARCHAR
                               auth_user_id BIGINT    ← 추가
                               (podo-auth JWT sub)
```

### 가족 공유 모델 (podo-bookshelf)

기존 podo-bookshelf는 `user_id` 필터로 개인별 데이터를 격리했습니다. 가족 도서관 개념으로 전환하면서 읽기와 쓰기의 정책을 다르게 가져갑니다.

- **읽기**: `user_id` 필터 제거 → 가족 전체가 모든 책 조회 가능
- **쓰기**: POST 시 `user_id` 기록 → 누가 추가했는지 추적

### 중앙 JWT 검증

3개 서비스 모두 동일한 `JWT_SECRET`을 공유하고, `issuer='podo-auth'` claim으로 토큰 출처를 검증합니다. 각 서비스는 podo-auth에 요청을 보내지 않고 로컬에서 토큰을 직접 검증합니다.
