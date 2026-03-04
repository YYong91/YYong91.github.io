---
title: "Billing 마이크로서비스"
description: "DDD 아키텍처 기반 결제·구독·크레딧 시스템"
date: 2026-03-04
tags: ["DDD", "CQRS", "gRPC", "Stripe", "Python", "CI/CD", "CodePipeline", "AWS"]
weight: 1
mermaid: true
---

## 개요

SaaS 플랫폼의 과금 체계를 담당하는 마이크로서비스입니다. Credit, Subscription, Payment 3개 Bounded Context를 DDD 패턴으로 설계하고 구현했습니다.

**기간**: 2026.01 ~ 현재
**소속**: 주식회사 큐빅 (CUBIG)

## 기술 스택

Python · SQLAlchemy · Alembic · gRPC · Stripe API · PostgreSQL · Pytest · AWS CodePipeline · CodeBuild · ECR

---

## 아키텍처

### Bounded Context 관계도

<div class="mermaid">
graph TB
subgraph "Billing Microservice"
subgraph "Credit BC"
CA[Credit Account]
CR[Credit Reservation]
CT[Credit Transaction]
end
subgraph "Subscription BC"
SP[Subscription Plan]
SS[Subscription Status]
end
subgraph "Payment BC"
PI[Payment Intent]
WH[Webhook Handler]
end
subgraph "Customer BC"
CU[Customer Aggregate]
PM[PG Mapping]
end
end
EXT_PG["External PG (Stripe)"]
EXT_ACC["Account Service (gRPC)"]
SS -->|"구독 갱신 시 크레딧 충전"| CA
WH -->|"결제 완료 → 구독 활성화"| SS
WH -->|"Webhook 수신"| EXT_PG
CU -->|"고객 식별"| CA
CU -->|"PG 매핑 소유"| PM
EXT_ACC -->|"gRPC: 고객 정보 조회"| CU
classDef bc fill:#faf6f1,stroke:#c96442,stroke-width:2px
classDef ext fill:#f0f0f0,stroke:#999,stroke-dasharray: 5 5
</div>

### 레이어 구조

<div class="mermaid">
graph LR
subgraph "Interface Layer"
API[REST API]
GRPC[gRPC Server]
end
subgraph "Application Layer"
CMD[Command Handlers]
QRY[Query Handlers]
end
subgraph "Domain Layer"
AGG[Aggregates]
EVT[Domain Events]
SVC[Domain Services]
end
subgraph "Infrastructure Layer"
REPO[Repositories]
DB[(PostgreSQL)]
MQ[Message Bus]
end
API --> CMD
API --> QRY
GRPC --> CMD
CMD --> AGG
CMD --> SVC
AGG --> EVT
QRY --> REPO
REPO --> DB
EVT --> MQ
</div>

### CI/CD 파이프라인 구조

<div class="mermaid">
graph LR
subgraph "PR Gate"
GH[GitHub PR]
CB_PR[CodeBuild Webhook<br/>FILE_PATH Filter]
TEST[pytest + coverage]
end
subgraph "Dev CD"
DEV_SRC[develop push]
CB_DEV[CodeBuild]
ECR_DEV[ECR]
CP_DEV[CodePipeline]
ECS_DEV[ECS Dev]
end
subgraph "Prod CI/CD"
PROD_SRC[main push]
CB_PROD[CodeBuild]
ECR_PROD[ECR]
CP_PROD[CodePipeline]
ECS_PROD[ECS Prod]
end
GH --> CB_PR
CB_PR --> TEST
DEV_SRC --> CB_DEV
CB_DEV --> ECR_DEV
ECR_DEV --> CP_DEV
CP_DEV --> ECS_DEV
PROD_SRC --> CB_PROD
CB_PROD --> ECR_PROD
ECR_PROD --> CP_PROD
CP_PROD --> ECS_PROD
</div>

---

## 주요 작업

- **2026-03-04**: Stripe API 2025+ breaking change 대응 — `subscription.current_period_*` 필드가 Subscription 객체에서 제거되어 `SubscriptionItem` 레벨로 이동. `construct_from()` 기반 SDK 테스트 패턴으로 전환 (233 test additions)
- **2026-03-04**: CI/CD 파이프라인 2-stage → 4/5-stage 분리 — PR gate(CodeBuild webhook) + Dev CD + Prod CI/CD로 역할 분리
- **2026-03-04**: 모노레포 FILE_PATH 필터 적용 — `apps/billing/**` 변경 시에만 파이프라인 트리거, 다른 서비스 PR에 의한 불필요한 빌드 차단
- **2026-03-04**: CI/CD 셋업 가이드 문서화 — 서비스별 재사용 가능한 파이프라인 설정 가이드 작성 (~500 lines)
- **2026-03-04**: Infisical → CodeBuild 환경변수 주입 설정 (buildspec.yml)
- **2026-02-19**: Customer Aggregate 도입으로 PG 매핑 소유권 도메인 내부로 이동 (99파일, 레이어별 커밋 분리)

---

## 주요 설계 결정

### 왜 3개 Bounded Context로 분리했는가

크레딧(선불), 구독(후불), 결제(PG 연동)는 각각 생명주기가 다릅니다. 크레딧은 예약→차감→환불의 상태 머신이 있고, 구독은 주기적 갱신 로직이 있으며, 결제는 외부 PG의 비동기 응답에 의존합니다. 하나의 Aggregate로 묶으면 상태 관리가 복잡해지기 때문에 BC 단위로 분리하고 Domain Events로 연결했습니다.

### Webhook 멱등성 처리

외부 PG는 동일한 이벤트를 여러 번 보낼 수 있습니다. Webhook Handler에서 event ID 기반 멱등성 키를 사용해 중복 처리를 방지했습니다. Payment 기록은 한 번만 저장되고, 이후 동일 요청은 무시됩니다.

### 크레딧 동시성 제어

여러 요청이 동시에 같은 크레딧 계좌에서 차감을 시도할 수 있습니다. Pessimistic row lock(`SELECT ... FOR UPDATE`)을 적용해 예약 시점에 잔액을 확보하고, 트랜잭션이 완료될 때까지 다른 요청은 대기하도록 했습니다.

### Customer Aggregate 도입

초기에는 외부 서비스의 조직 ID를 직접 참조했습니다. PG 매핑 정보가 도메인 외부에 의존하는 구조가 되면서, Customer Aggregate를 도입해 PG 매핑 소유권을 도메인 내부로 이동했습니다. 이 과정에서 99개 파일을 레이어별로 단계적 마이그레이션했습니다.

### gRPC 선택 이유

마이크로서비스 간 통신에서 REST 대비 gRPC를 선택한 이유는 스키마 강제(proto)와 타입 안전성 때문입니다. Billing ↔ Account 간 고객 정보 조회가 빈번한데, proto 파일이 곧 API 계약이 되어 서비스 간 인터페이스 변경을 추적하기 쉬웠습니다.

### CodeBuild Webhook을 PR gate로 사용

CodePipeline은 브랜치 푸시 트리거만 지원해 PR 단계에서 테스트를 실행할 수 없었습니다. CodeBuild Webhook을 별도로 설정해 PR open/update 이벤트에 테스트를 실행하는 gate를 만들었습니다. CodePipeline은 develop/main 브랜치 머지 후 배포만 담당하는 역할 분리를 적용했습니다.

### 모노레포 FILE_PATH 필터

billing 서비스 변경이 없는데도 다른 서비스 PR이 billing 파이프라인을 트리거하는 문제가 있었습니다. CodeBuild Webhook의 `FILE_PATH` 필터를 `apps/billing/**`로 제한해 billing 코드가 변경된 경우에만 CI가 실행되도록 했습니다.

### Stripe SDK 테스트 패턴 전환

Stripe API 2025-01-27 버전부터 `subscription.current_period_start/end`가 Subscription 레벨에서 제거되고 `SubscriptionItem` 레벨로 이동했습니다. 기존 테스트는 dict로 Stripe 객체를 직접 구성했기 때문에 런타임에서만 오류가 발생했습니다. `stripe.Subscription.construct_from()` 방식으로 전환해 SDK가 검증하는 실제 객체 구조로 테스트하도록 변경했습니다.

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 객체 생성 | `{"current_period_end": ...}` dict | `stripe.Subscription.construct_from({...})` |
| 검증 시점 | 런타임 실패 | 테스트 단계에서 구조 불일치 감지 |
| API 버전 대응 | 수동 dict 키 관리 | SDK 스키마 자동 반영 |

---

## 테스트 전략

| 레벨 | 대상 | 방식 |
|------|------|------|
| 단위 테스트 | Domain Layer (Aggregate, Value Object) | 순수 Python, 외부 의존 없음 |
| 서비스 테스트 | Application Layer (Command/Query Handler) | Fake Repository 주입 |
| 통합 테스트 | Infrastructure Layer (Repository, DB) | Testcontainers + PostgreSQL |
| E2E 테스트 | 전체 비즈니스 시나리오 | 구독 생성 → 결제 → 크레딧 충전 → 차감 흐름 |
