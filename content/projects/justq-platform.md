---
title: "멀티채널 상품 관리 플랫폼"
description: "DDD, MSA, RabbitMQ 기반 대규모 비동기 처리 시스템"
date: 2020-07-01
tags: ["DDD", "MSA", "RabbitMQ", "Python", "AWS", "Terraform", "Multi-Tenancy"]
weight: 10
mermaid: true
---

## 개요

커머스 데이터를 여러 마켓플레이스에 제공하는 B2B 시스템을 개발하고 운영했습니다. 대규모 데이터를 안정적으로 처리하는 구조를 만드는 데 집중했습니다.

**기간**: 2020.07 ~ 2025.12
**소속**: 주식회사 저스트큐 (JustQ)

## 기술 스택

Python · SQLAlchemy · Alembic · RabbitMQ (AMQP) · ActiveMQ · MySQL · AWS (ECS, Fargate, Lambda) · Docker · Terraform · Jenkins · GitHub Actions · CloudWatch · Datadog

---

## 아키텍처

### MSA 전체 구조

<div class="mermaid">
graph TB
subgraph "API Gateway"
GW[API Gateway]
end
subgraph "Core Services"
PS[Product Service]
OS[Order Service]
IS[Inventory Service]
end
subgraph "Integration Layer"
SYNC[Sync Worker]
EXT[External API Adapter]
end
subgraph "Messaging"
MQ[RabbitMQ Cluster]
DLQ[Dead Letter Queue]
end
subgraph "Infrastructure"
ECS[ECS / Fargate]
LMB[Lambda]
DB[(MySQL)]
MON[Datadog / CloudWatch]
end
GW --> PS
GW --> OS
GW --> IS
PS -->|"상품 변경 이벤트"| MQ
OS -->|"주문 이벤트"| MQ
MQ --> SYNC
MQ --> DLQ
SYNC --> EXT
EXT -->|"외부 마켓 API"| ECS
PS --> DB
OS --> DB
IS --> DB
LMB -->|"MQ 메트릭 기반 Auto Scaling"| ECS
MON --> LMB
</div>

### Multi-Tenancy 구조

<div class="mermaid">
graph LR
subgraph "Shared Layer"
API[공통 API]
AUTH[인증/인가]
CORE[공통 도메인 로직]
end
subgraph "Tenant Isolation"
T1[Tenant A Config]
T2[Tenant B Config]
T3[Tenant C Config]
end
subgraph "Data Layer"
DB[(Shared DB)]
SCH1[Schema A]
SCH2[Schema B]
SCH3[Schema C]
end
API --> AUTH
AUTH -->|"tenant_id 주입"| CORE
CORE --> T1
CORE --> T2
CORE --> T3
T1 --> SCH1
T2 --> SCH2
T3 --> SCH3
SCH1 --> DB
SCH2 --> DB
SCH3 --> DB
</div>

### RabbitMQ 메시징 흐름

<div class="mermaid">
graph LR
PUB[Publisher]
EX[Exchange]
subgraph "Queue Layer"
Q1[product.sync]
Q2[order.process]
Q3[inventory.update]
end
subgraph "Error Handling"
RETRY[Retry Queue]
DLQ[Dead Letter Queue]
ALERT[Alert]
end
CON[Consumer Workers]
PUB -->|"routing key"| EX
EX --> Q1
EX --> Q2
EX --> Q3
Q1 --> CON
Q2 --> CON
Q3 --> CON
CON -->|"처리 실패"| RETRY
RETRY -->|"3회 재시도"| CON
RETRY -->|"최종 실패"| DLQ
DLQ --> ALERT
</div>

---

## 주요 설계 결정

### 🏗 왜 DDD를 도입했는가

초기에는 레이어드 아키텍처에 서비스 클래스가 비대해지는 구조였습니다. 상품·주문·재고가 하나의 서비스에서 얽히면서, 기능 하나를 변경하면 의존 범위를 파악하기 어려워졌습니다. 도메인을 분리하고 팀 간 용어를 통일한 뒤, 비즈니스 규칙을 도메인 모델에 집중시켜 변경 영향 범위를 줄였습니다.

### 📨 ActiveMQ → RabbitMQ 전환

ActiveMQ에서 메시지 유실과 성능 한계가 반복되었습니다. RabbitMQ로 전환하면서 Routing Key 전략, DLQ, 재시도 정책을 함께 설계했습니다. 전환 후 시간당 30만 건, 하루 600만 건 이상의 메시지를 안정적으로 처리할 수 있게 되었습니다.

### 🏢 Multi-Tenancy 선택 이유

고객사별로 별도 인스턴스를 운영하면 인프라 비용과 관리 부담이 선형으로 증가합니다. Shared DB + Schema 분리 전략을 선택해 공통 로직은 재사용하고, 고객사별 설정과 데이터만 격리했습니다. 신규 고객 온보딩 시간이 크게 줄었고, 유지보수 비용을 약 25% 절감했습니다.

### ⚡ MQ 이벤트 기반 Auto Scaling

CloudWatch에서 RabbitMQ 큐 깊이 메트릭을 수집하고, Lambda가 ECS 서비스의 desired count를 조정하는 구조를 만들었습니다. 트래픽이 몰리는 시간대에만 워커가 스케일 아웃되고, 유휴 시간에는 최소 인스턴스로 운영해 운영 비용을 약 35% 절감했습니다.

### 🔒 멱등성 보장

외부 마켓 API는 응답이 불안정할 수 있습니다. 메시지 처리 시 idempotency key를 기록하고, 동일 메시지가 재전달되어도 중복 처리가 발생하지 않도록 했습니다. DLQ로 빠진 메시지는 별도 알림을 통해 수동 복구 또는 자동 재처리 판단을 하는 구조입니다.

---

## 성과

| 항목 | 수치 |
|------|------|
| 메시지 처리량 | 시간당 30만 건, 하루 600만+ 건 |
| 운영 비용 절감 | Auto Scaling으로 약 35% |
| 유지보수 비용 절감 | Multi-Tenancy 도입으로 약 25% |
| 외부 연동 오류율 | 재시도/멱등성 도입 후 약 30% 감소 |
