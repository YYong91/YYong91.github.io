---
title: "[ADR-001] Bounded Context를 3개로 분리한 이유"
date: 2026-02-20
categories: ["ddd"]
tags: ["DDD", "Bounded Context", "MSA", "ADR", "아키텍처"]
series: ["Billing 마이크로서비스 ADR"]
series_order: 1
---

> **ADR(Architecture Decision Record)**: 아키텍처 결정과 그 이유를 기록하는 문서입니다. Billing 마이크로서비스를 설계하면서 내린 결정들을 시리즈로 정리합니다.

---

## 상황 (Context)

Billing 마이크로서비스를 설계할 때 첫 번째 질문은 "어떤 단위로 도메인을 나눌 것인가"였습니다.

과금 도메인에는 크게 세 가지 개념이 있습니다:

- **크레딧**: 선불로 충전한 잔액. 예약·차감·환불의 상태 머신
- **구독**: 주기적 갱신. 플랜, 활성화 여부, 갱신 주기 관리
- **결제**: 외부 PG(Payment Gateway)와의 연동. 비동기 Webhook 처리

초기 설계안은 이 세 가지를 하나의 도메인으로 묶는 것이었습니다. "결국 다 돈 관련이니까"라는 논리였습니다.

---

## 문제 (Problem)

하나로 묶었을 때 생기는 문제는 설계를 진행하면서 드러났습니다.

| 개념 | 생명주기 | 변경 트리거 | 외부 의존 |
|------|---------|-----------|----------|
| 크레딧 | 예약 → 차감 → 환불 | 사용자 액션 | 없음 |
| 구독 | 활성 → 갱신 → 만료 | 시간(스케줄러) | 크레딧 시스템 |
| 결제 | 요청 → 대기 → 완료/실패 | 외부 PG Webhook | Stripe API |

생명주기가 전부 다릅니다. 크레딧은 사용자 요청에 반응하고, 구독은 시간에 반응하고, 결제는 외부 이벤트에 반응합니다. 하나의 Aggregate에 넣으면:

- 크레딧 차감 로직이 구독 갱신 스케줄러를 건드림
- 결제 Webhook 처리가 크레딧 잔액 조회와 트랜잭션을 공유함
- 어디서 변경이 일어나도 전체 Aggregate가 로드됨

결국 "하나의 Aggregate"는 모든 결제 관련 개념을 담은 God Object가 됩니다.

---

## 결정 (Decision)

**Credit, Subscription, Payment를 각각 독립적인 Bounded Context로 분리합니다.**

```
Billing Microservice
├── Credit BC       (크레딧 계좌, 예약, 트랜잭션)
├── Subscription BC (플랜, 구독 상태)
├── Payment BC      (결제 의도, Webhook 처리)
└── Customer BC     (PG 매핑 소유권 → ADR-002 참고)
```

BC 간 통신은 Domain Event로 처리합니다:
- Webhook 수신 → Payment BC → `PaymentCompleted` 이벤트 → Subscription BC 활성화
- 구독 갱신 → Subscription BC → `SubscriptionRenewed` 이벤트 → Credit BC 충전

---

## 이유 (Rationale)

**대안 1: 단일 Aggregate**

가장 단순한 구조지만, 생명주기가 다른 세 개념을 한 곳에 묶으면 변경 이유가 세 가지가 됩니다. SRP(단일 책임 원칙) 위반입니다. 크레딧 로직 변경이 결제 코드를 건드릴 이유가 없습니다.

**대안 2: 레이어드 아키텍처 + 서비스 분리**

BC 없이 Service 클래스로 분리하는 방법도 있습니다. 하지만 서비스 클래스는 도메인 규칙의 경계를 강제하지 않습니다. 시간이 지나면 `CreditService`가 `SubscriptionRepository`를 직접 호출하는 코드가 생깁니다.

**BC 분리를 선택한 이유:**

1. 생명주기가 다른 개념은 트랜잭션 경계도 달라야 합니다
2. 크레딧 동시성 문제(비관적 락)를 결제 처리와 무관하게 격리할 수 있습니다
3. 나중에 Credit만 별도 서비스로 추출할 수 있는 구조가 됩니다

---

## 결과 (Outcome)

BC 분리 후 각 컨텍스트가 자신의 Aggregate, Repository, Domain Event를 독립적으로 가집니다.

트레이드오프도 있습니다:

| 장점 | 단점 |
|------|------|
| 각 BC가 독립적으로 변경 가능 | BC 간 조율 로직(Domain Event) 필요 |
| 트랜잭션 경계가 명확 | 분산 트랜잭션 상황 고려 필요 |
| 동시성 문제를 BC 안에서 격리 | 코드 양이 늘어남 |

"생명주기가 다르면 도메인도 다르다"는 기준이 BC 분리의 핵심 판단 근거였습니다.
