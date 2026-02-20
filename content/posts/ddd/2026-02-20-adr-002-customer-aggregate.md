---
title: "[ADR-002] Customer Aggregate 도입 — 외부 의존을 도메인 안으로"
date: 2026-02-20
categories: ["ddd"]
tags: ["DDD", "Aggregate", "ADR", "아키텍처", "리팩토링"]
series: ["Billing 마이크로서비스 ADR"]
series_order: 2
---

> **이 시리즈**: Billing 마이크로서비스 설계 결정 기록. [ADR-001: BC 분리 기준](/posts/ddd/2026-02-20-adr-001-bounded-context/)에서 이어집니다.

---

## 상황 (Context)

초기 설계에서 Billing 서비스는 고객을 직접 관리하지 않았습니다. 고객 정보(조직 ID)는 Account 서비스가 소유하고, Billing은 그 ID를 그대로 참조하는 구조였습니다.

```python
# 초기 구조
class CreditAccount:
    organization_id: str  # Account 서비스의 외부 ID를 직접 사용
    balance: Decimal
```

Credit 계좌를 만들 때, 구독을 처리할 때, 결제를 연동할 때마다 `organization_id`를 키로 Account 서비스를 조회했습니다.

---

## 문제 (Problem)

PG 연동을 추가하면서 문제가 드러났습니다.

Stripe 같은 외부 PG는 고객마다 고유한 `customer_id`를 발급합니다. 이 매핑 정보(`organization_id` ↔ `stripe_customer_id`)를 어디에 저장할지 결정해야 했습니다.

**선택지 분석:**

| 선택 | 문제점 |
|------|--------|
| Account 서비스에 저장 | Account 서비스가 PG 매핑을 알아야 함. 도메인 책임이 섞임 |
| Billing에서 `organization_id`로 직접 관리 | PG 매핑 소유자가 없음. 어느 BC가 관리하는지 불명확 |
| Billing에 Customer Aggregate 도입 | PG 매핑 소유권이 명확해짐 |

`organization_id`를 직접 쓰는 구조의 핵심 문제는 **PG 매핑의 소유권이 도메인 밖에 있다**는 점이었습니다. Billing 도메인이 결제를 처리하려면 항상 외부 서비스에 의존해야 합니다.

---

## 결정 (Decision)

**Billing 도메인 안에 Customer Aggregate를 도입합니다.**

```python
# 변경 후 구조
class Customer:  # Billing 도메인의 Customer
    id: CustomerId          # Billing 내부 ID
    external_account_id: str  # Account 서비스 참조 (읽기 전용)
    pg_mappings: list[PGMapping]  # PG 매핑 소유

class PGMapping:
    provider: str           # "stripe", "toss" 등
    external_customer_id: str  # PG에서 발급한 customer_id
```

Customer Aggregate가 PG 매핑을 소유합니다. Account 서비스의 ID는 참조용으로만 보관하고, 결제 관련 모든 정보는 Billing 도메인 안에서 관리합니다.

---

## 이유 (Rationale)

**PG 매핑은 누가 소유해야 하는가**

PG 매핑은 결제를 위해 존재합니다. 결제는 Billing 도메인의 책임입니다. 따라서 PG 매핑의 소유권도 Billing에 있어야 합니다. Account 서비스에 두는 건 책임을 잘못 배치하는 것입니다.

**외부 ID를 직접 참조하면 생기는 문제:**

도메인 객체 안에 외부 서비스의 ID가 직접 들어오면, 그 ID의 의미를 Billing 도메인이 알 수 없습니다. `organization_id`가 "어떤 계층의 ID인지", "하나의 조직에 여러 PG 매핑이 가능한지" 같은 규칙이 도메인 코드 밖에 암묵적으로 존재하게 됩니다.

---

## 결과 (Outcome)

**마이그레이션 규모**: 99개 파일

`organization_id` 직접 참조를 `CustomerId`로 교체하는 작업이 전체 Billing 서비스에 걸쳐 있었습니다. 레이어별로 단계적으로 진행했습니다:

1. Domain Layer: Customer Aggregate, PGMapping Value Object 정의
2. Infrastructure Layer: CustomerRepository, PGMapping 테이블 추가
3. Application Layer: Command/Query Handler에서 `organization_id` → `customer_id` 교체
4. Interface Layer: API 응답 스키마 수정

각 레이어를 완료하고 테스트를 통과시킨 뒤 다음 레이어로 진행했습니다. 한 번에 전체를 바꾸지 않고 레이어 단위로 묶어서 중간에 빌드가 깨지더라도 범위를 파악할 수 있도록 했습니다.

**트레이드오프:**

| 장점 | 단점 |
|------|------|
| PG 매핑 소유권이 명확 | Customer 생성 시 Account 서비스와 동기화 필요 |
| Billing 도메인이 완결성을 가짐 | 마이그레이션 비용(99개 파일) |
| 여러 PG 매핑 지원이 자연스러워짐 | Customer 개념이 하나 더 생김(Account vs Billing Customer) |

99개 파일 마이그레이션은 비쌌지만, 이후 새로운 PG를 추가할 때 Account 서비스를 건드리지 않아도 되는 구조가 되었습니다.
