---
title: "3개 레포를 동시에 분석해서 SSO 아키텍처 결정을 내린 과정"
date: 2026-02-21
categories: ["claude-code"]
tags: ["claude-code", "architecture", "SSO", "JWT", "brainstorming", "writing-plans", "claude-opus"]
---

podo-budget에 SSO를 붙이는 작업을 했습니다. podo-auth, podo-bookshelf, podo-budget — 세 서비스가 얽혀 있어서, 코드를 직접 다 읽고 설계를 잡으려면 꽤 오래 걸릴 상황이었습니다.

---

## 📌 상황

podo-bookshelf는 이미 SSO가 붙어 있습니다. podo-budget은 자체 인증(self-auth)을 쓰고 있습니다. 두 서비스를 같은 podo-auth JWT로 묶으려면 먼저 각 서비스의 인증 흐름과 DB 스키마 관계를 파악해야 했습니다.

문제는 FK 관계와 PK 타입이 복잡하게 얽혀 있다는 점이었습니다. podo-auth의 user ID는 `BigInt`, podo-budget의 `users.id`는 `Integer`입니다. 그리고 budget 쪽 `users` 테이블은 7개 도메인 모델의 FK 참조를 받고 있습니다.

---

## 🔍 분석: Opus 모델로 3개 레포 동시 탐색

`brainstorming` 스킬을 열고, Claude Opus 4.6에게 세 레포를 동시에 넘겼습니다.

분석 대상:
- `podo-auth`: JWT 발급 구조, `sub` 클레임이 username인지 user_id인지
- `podo-bookshelf`: 이미 SSO가 어떻게 붙어 있는지, JWT 검증 미들웨어 구조
- `podo-budget`: SQLAlchemy 모델 전체, FK 관계, `users` 테이블 의존성

Opus가 반환한 핵심 발견 사항들:

| 발견 | 내용 |
|------|------|
| JWT sub 클레임 | `sub=username` (auth_user_id가 아님) — bookshelf에서 확인 |
| PK 타입 충돌 | `auth.users.id = BigInt` vs `budget.users.id = Integer` |
| FK 의존 모델 수 | budget `users` 테이블을 참조하는 도메인 모델 7개 |
| bookshelf SSO 패턴 | issuer 클레임 검증 후 자체 user 테이블과 매핑 |

이 정보를 직접 읽어서 정리했다면, 레포 3개를 오가며 auth 흐름 → JWT 구조 → SQLAlchemy 모델 → 마이그레이션 영향 순서로 추적해야 합니다. 반나절은 잡아야 했을 작업입니다.

---

## 🏗️ 아키텍처 결정: 분석 결과가 설계로

Opus 분석 결과를 받고 `writing-plans` 스킬로 구현 계획을 잡았습니다. 분석 전에는 "일단 users.id를 BigInt로 바꾸면 되지 않나" 싶었지만, 7개 도메인 모델이 FK로 물려 있다는 걸 보고 방향을 바꿨습니다.

**결정 1: Shadow User 패턴**

`budget.users` 테이블의 Integer PK는 건드리지 않습니다. 대신 `auth_user_id BigInteger` 컬럼을 추가해서 podo-auth JWT의 user를 매핑하는 방식입니다. 기존 7개 도메인 모델의 FK는 그대로 유지됩니다.

마이그레이션 범위가 7개 모델 전체 vs 컬럼 하나 추가 — 분석이 없었으면 잘못된 선택을 했을 가능성이 높습니다.

**결정 2: 가족 공유 모델**

podo-budget의 사용 방식을 분석하다가 나온 결정입니다. 가족이 함께 쓰는 가계부이기 때문에, 읽기는 `user_id` 필터 없이 공유하고 쓰기만 작성자를 추적하는 구조가 맞았습니다. 코드 분석을 통해 실제 사용 패턴이 드러난 결과입니다.

**결정 3: JWT issuer 클레임 검증**

세 서비스 모두 `issuer='podo-auth'`를 검증합니다. bookshelf 코드에서 이미 이 패턴이 있었고, budget도 같은 방식으로 적용하는 걸로 결정했습니다. 다른 서비스의 JWT가 budget에 사용되는 상황을 방지합니다.

---

## ✔ 실제로 어땠는지

잘 된 부분은 분석과 설계의 연결입니다. "Integer vs BigInt PK 충돌이 있다"는 발견이 바로 "Shadow User 패턴을 쓴다"는 결정으로 이어졌습니다. 코드 분석 → 제약 조건 파악 → 아키텍처 선택의 흐름이 자연스러웠습니다.

어색했던 부분은 Opus 모델을 쓰다 보니 분석 속도가 느리다는 점이었습니다. 3개 레포를 동시에 넘기면 응답까지 시간이 걸립니다. 빠르게 방향만 잡아야 할 때는 Sonnet으로 먼저 큰 그림을 보고, 정밀 분석이 필요할 때 Opus로 넘기는 방식이 더 나을 것 같습니다.

`writing-plans`로 나온 계획서는 마이그레이션 단계까지 포함되어 있어서, 다음 세션에서 어디부터 시작해야 할지 명확합니다. `brainstorming` + `writing-plans` 조합을 설계 세션에서 쓰는 게 점점 루틴이 되고 있습니다.
