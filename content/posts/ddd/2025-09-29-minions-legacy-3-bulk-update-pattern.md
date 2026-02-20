---
title: "미니언즈 레거시(3) - 대량 업데이트 처리 시 어색했던 패턴"
date: 2025-09-29
categories: ["ddd"]
tags: ["DDD", "DomainEvent", "IntegrationEvent"]
series: ["미니언즈 레거시"]
migrated_from: "velog"
---

## 주제

상품 대량 업데이트 정보를 수신하면, 각 Product Aggregate마다 단건 데이터를 수정하고 트랜잭션 종료 후 외부 시스템으로 전달하기 위해 하나의 Integration Event를 발행해야 합니다. 그러나 단건 Domain Event를 그대로 발생시키거나 Application Layer에서 억지로 묶어 이벤트를 만들면 어색한 패턴이 나타납니다. 본 글은 이러한 어색함이 왜 발생하는지, 예시와 함께 살펴보고 대안들을 개괄적으로 정리합니다.

---

## 문제 상황: 대량 업데이트를 처리할 때

Domain Event는 원칙적으로 **Domain Layer 내부에서 발생**하는 것이 자연스럽습니다. 단건 처리라면 다음과 같이 깔끔합니다.

```python
class Product(AggregateRoot):
    def update_status(self, new_status: str):
        self.status = new_status
        self.add_domain_event(ProductStatusUpdatedDomainEvent(product_id=self.id))
```

하지만 상품 대량 업데이트 정보를 수신해 여러 Product를 동시에 수정한 뒤, 외부 시스템에는 **하나의 Integration Event**로 묶어 발행하고자 할 때 문제가 발생했습니다.

### 단건 이벤트를 그대로 쓰기 어려운 이유

* 각 Product가 개별적으로 `ProductStatusUpdatedDomainEvent`를 발생시키면, 수십·수백 개의 이벤트가 흩어져 외부 시스템에 전달됩니다.
* 외부 시스템은 하나의 대량 업데이트 이벤트를 기대하기 때문에, 단건 이벤트를 그대로 퍼블리싱하면 불필요한 호출과 중복된 데이터 흐름이 발생합니다.
* 결국 **트랜잭션 종료 시점에 다건을 묶어 하나의 Integration Event로 발행해야 하는 요구**가 생깁니다.

### 실제로 했던 방식

```python
# Application Layer에서의 임시 방편
updated_products[0].add_domain_event(
    ProductsStatusUpdatedIntegrationEvent(aggregates=updated_products)
)
```

그리고 `message_sender` 클래스에서 이 이벤트를 핸들링하여 최종 Integration Event를 만들어 퍼블리시했습니다.

---

## 왜 어색해졌는가

1. **이벤트 발생 위치가 어긋남**
   Domain Layer 내부에서 발생해야 할 이벤트를 Application Layer에서 억지로 삽입했습니다.

2. **대표 Aggregate에 대리 등록**
   왜 하필 `updated_products[0]`인지 불분명합니다. 이벤트 소유권이 모호해졌습니다.

3. **테스트 경계 불명확**
   원래는 Domain Layer 단위 테스트로 충분했지만, 이제는 Application Layer 단위 테스트로 이벤트를 검증해야 합니다.

4. **Integration Event만을 위한 Domain Event**
   다건 이벤트는 외부 발행만을 위한 것으로, Domain Layer 내에서는 의미가 없습니다.

   * 반면 단건의 `ProductStatusUpdatedDomainEvent`는 동일 서비스 내 다른 Aggregate가 이 이벤트를 핸들링하면서 새로운 로직을 구성할 수 있습니다.
   * 하지만 다건 이벤트는 재사용성이 없고, Domain Layer의 표현력에도 기여하지 못합니다.

5. **경계 분리가 무너짐**
   Domain Event와 Integration Event를 억지로 섞으면서, 원칙이었던 **Domain Layer에서 의미를 충분히 담고, 다른 영역은 이를 활용만 하는 구조**가 깨졌습니다.

---

## 가능한 대안들 (개괄)

### 1. 단건 이벤트 유지, Application Layer에서 Integration Event 조립

* 각 Aggregate는 자기 Domain Event만 발생시킵니다.
* Application Layer 또는 Publisher는 트랜잭션 종료 후 Outbox에 기록된 단건 이벤트들을 모아 Integration Event를 생성합니다.
* 장점: Domain Layer의 순수성 유지, 테스트 경계 명확.
* 단점: 조립 로직이 필요해 초기에는 번거로울 수 있습니다.

### 2. Domain Service 도입

* 대량 업데이트 자체를 하나의 비즈니스 행위로 모델링합니다.
* Domain Service가 여러 Aggregate를 순회하며 이벤트를 발생시킵니다.
* 장점: 묶음이 실제 도메인 개념일 경우 유용합니다.
* 단점: 단순 반복문 래퍼로만 남으면 불필요한 복잡도를 초래합니다.

### 3. 소비자 측 묶기(윈도우링/버퍼링)

* 발행은 단건 Domain Event 그대로 두고, 컨슈머가 일정 조건에 따라 묶어 처리합니다.
* 장점: 발행 측 단순, 유연한 묶음 처리 가능.
* 단점: 즉시 묶음 보장이 어렵고, 복잡도가 소비자 측으로 전가됩니다.

---

## 참고할 점

* **Domain Event와 Integration Event의 역할 구분**: Domain Event는 상태 변화를 나타내고, Integration Event는 외부 통합을 위해 사용됩니다.
* **Outbox 패턴**: 트랜잭션과 이벤트 발행을 정합성 있게 연결하는 핵심 기법입니다.
* **테스트 전략**: Domain Event는 단위 테스트에서 검증하고, Integration Event는 통합 테스트에서 검증하는 것이 바람직합니다.

---

## 다음 글 예고

이번 글에서는 대량 업데이트 시 어색해지는 패턴과 그 원인, 대안 개괄을 정리했습니다. 다음 글에서는 Domain Service를 활용하여 다건 이벤트를 도메인적으로도 의미 있게 처리하는 리팩터링 사례를 코드와 함께 소개할 예정입니다.
