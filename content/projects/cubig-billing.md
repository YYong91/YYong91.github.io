---
title: "Billing 마이크로서비스"
description: "DDD 아키텍처 기반 결제·구독·크레딧 시스템"
date: 2026-01-28
tags: ["DDD", "CQRS", "gRPC", "Stripe", "Python"]
weight: 1
mermaid: true
---

## 개요

SaaS 플랫폼의 과금 체계를 담당하는 마이크로서비스입니다. Credit, Subscription, Payment 3개 Bounded Context를 DDD 패턴으로 설계하고 구현했습니다.

**기간**: 2026.01 ~ 현재
**소속**: 주식회사 큐빅 (CUBIG)

## 기술 스택

Python · SQLAlchemy · Alembic · gRPC · Stripe API · PostgreSQL · Pytest

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

---

## 주요 설계 결정

### 🏗 왜 3개 Bounded Context로 분리했는가

크레딧(선불), 구독(후불), 결제(PG 연동)는 각각 생명주기가 다릅니다. 크레딧은 예약→차감→환불의 상태 머신이 있고, 구독은 주기적 갱신 로직이 있으며, 결제는 외부 PG의 비동기 응답에 의존합니다. 하나의 Aggregate로 묶으면 상태 관리가 복잡해지기 때문에 BC 단위로 분리하고 Domain Events로 연결했습니다.

### 💳 Webhook 멱등성 처리

외부 PG는 동일한 이벤트를 여러 번 보낼 수 있습니다. Webhook Handler에서 event ID 기반 멱등성 키를 사용해 중복 처리를 방지했습니다. Payment 기록은 한 번만 저장되고, 이후 동일 요청은 무시됩니다.

### 🔒 크레딧 동시성 제어

여러 요청이 동시에 같은 크레딧 계좌에서 차감을 시도할 수 있습니다. Pessimistic row lock(`SELECT ... FOR UPDATE`)을 적용해 예약 시점에 잔액을 확보하고, 트랜잭션이 완료될 때까지 다른 요청은 대기하도록 했습니다.

### 🔄 Customer Aggregate 도입

초기에는 외부 서비스의 조직 ID를 직접 참조했습니다. PG 매핑 정보가 도메인 외부에 의존하는 구조가 되면서, Customer Aggregate를 도입해 PG 매핑 소유권을 도메인 내부로 이동했습니다. 이 과정에서 99개 파일을 레이어별로 단계적 마이그레이션했습니다.

### 📡 gRPC 선택 이유

마이크로서비스 간 통신에서 REST 대비 gRPC를 선택한 이유는 스키마 강제(proto)와 타입 안전성 때문입니다. Billing ↔ Account 간 고객 정보 조회가 빈번한데, proto 파일이 곧 API 계약이 되어 서비스 간 인터페이스 변경을 추적하기 쉬웠습니다.

---

## 테스트 전략

| 레벨 | 대상 | 방식 |
|------|------|------|
| 단위 테스트 | Domain Layer (Aggregate, Value Object) | 순수 Python, 외부 의존 없음 |
| 서비스 테스트 | Application Layer (Command/Query Handler) | Fake Repository 주입 |
| 통합 테스트 | Infrastructure Layer (Repository, DB) | Testcontainers + PostgreSQL |
| E2E 테스트 | 전체 비즈니스 시나리오 | 구독 생성 → 결제 → 크레딧 충전 → 차감 흐름 |
