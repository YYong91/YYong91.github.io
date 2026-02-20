---
title: "미니언즈 레거시(2) - 도메인 이벤트와 어그리게이트(단건 트랜잭션 패턴)"
date: 2025-09-24
categories: ["ddd"]
tags: ["DDD", "도메인 이벤트"]
series: ["미니언즈 레거시"]
migrated_from: "velog"
---

마이크로서비스 환경에서 트랜잭션은 단순히 데이터베이스 커밋으로 끝나지 않습니다. API 요청이나 MQ 메시지 수신으로 시작된 작업은 **어그리게이트의 도메인 메서드 → 도메인 이벤트 발행 → 이벤트 핸들링**을 거쳐 다음 단계로 이어집니다.

이번 글(1편)에서는 **도메인 서비스 없이**, **Application Layer가 어그리게이트의 도메인 메서드를 호출**하고, 어그리게이트가 **도메인 이벤트를 발행**하며, 각 **behavior 모듈**이 이벤트를 구독해 다음 단계를 실행하는 **단건 트랜잭션 기본 흐름**을 정리합니다. (2편에서 다중 어그리게이트와 도메인 서비스 패턴을 다룹니다.)

---

## 어그리게이트와 도메인 이벤트

**어그리게이트(Aggregate)**

- 여러 엔티티와 값 객체를 묶어 **일관성**을 보장하는 단위
- 모든 상태 변경은 **루트 엔티티의 도메인 메서드**로만 수행
- 원자성/불변 비즈니스 규칙을 내부에서 보장

**도메인 이벤트(Domain Event)**

- **이미 발생한 비즈니스적으로 중요한 사건**을 표현 (보통 과거 시제: `OrderPlaced`, `PaymentCompleted`)
- 어그리게이트 간 **직접 참조를 피하고 느슨한 결합**을 유지
- 이벤트 핸들러가 후속 로직을 수행하거나, 어댑터를 통해 외부 시스템과 통합

---

## 트랜잭션 과정: 기본 흐름(단건)

비즈니스 시나리오로 살펴보면 다음과 같습니다.

- 사용자가 쇼핑몰에서 주문 버튼을 누릅니다. (Order API 호출)
- 주문이 생성되면 Order Aggregate는 상태를 `PLACED`로 변경하고 `OrderPlacedDomainEvent`를 발행합니다.
- `payment_behavior.py`는 이 이벤트를 구독해 PG 결제 프로세스를 실행하고, 결제가 성공하면 `PaymentCompletedDomainEvent`를 발행합니다.
- `shipping_behavior.py`는 결제 완료 이벤트를 구독해 배송 절차를 시작하고, `ShippingInitiatedDomainEvent`를 발행합니다.
- 마지막으로 MQ Publisher가 이 이벤트를 외부 재고 관리 서비스 등과 동기화하기 위해 MQ로 송신합니다.

즉, 주문 생성 → 결제 완료 → 배송 시작 → 외부 시스템 동기화까지의 전체 과정이 **도메인 이벤트 체인**으로 연결됩니다.

1. **Application Layer에서 시작**
   사용자가 **Order API**를 호출하면 Application Layer의 **Order API Handler**가 실행되고, **Order Aggregate의 도메인 메서드**를 호출합니다.

2. **어그리게이트 상태 변경 & 이벤트 발행**
   Order Aggregate는 상태를 변경한 뒤 **`OrderPlacedDomainEvent`**를 발행합니다.

3. **`payment_behavior.py`에서 처리**
   `OrderPlacedDomainEvent`를 **payment_behavior.py**가 구독하여 **Payment Aggregate의 도메인 메서드**를 수행하고, 결제가 성공하면 **`PaymentCompletedDomainEvent`**를 발행합니다.

4. **`shipping_behavior.py`에서 처리**
   `PaymentCompletedDomainEvent`를 **shipping_behavior.py**가 구독하여 **Shipping Aggregate의 도메인 메서드**를 수행하고, 배송을 시작하며 **`ShippingInitiatedDomainEvent`**를 발행합니다.

5. **외부 통신 (Adapter/MQ)**
   `ShippingInitiatedDomainEvent`는 **MQ Publisher(어댑터)**를 통해 MQ로 송신되어 외부(예: 재고 관리 시스템)와 동기화됩니다.

![트랜잭션 단건 흐름](/images/domain-event-single-transaction-flow.png)

---

## 간단한 코드 예시 (Python)

```python
from dataclasses import dataclass

# Events
@dataclass
class OrderPlacedDomainEvent:
    order_id: str

@dataclass
class PaymentCompletedDomainEvent:
    order_id: str

# Aggregates
class Order:
    def __init__(self, order_id: str):
        self.id = order_id
        self.status = "INIT"

    def place(self):
        # 도메인 규칙 검증 및 상태 변경
        self.status = "PLACED"
        # 상태 변경 이후 도메인 이벤트 발행
        return OrderPlacedDomainEvent(order_id=self.id)

class Payment:
    def complete(self, order_id: str):
        # 결제 로직 ... (ex. PG 승인/검증)
        # 결제 완료 이벤트 생성
        return PaymentCompletedDomainEvent(order_id=order_id)

class Shipping:
    def initiate(self, order_id: str):
        # 배송 시작 로직 ...
        return {"event": "ShippingInitiatedDomainEvent", "order_id": order_id}

# Application Layer (Order API Handler)
def place_order_api(order_id: str, event_bus):
    order = Order(order_id)
    event = order.place()                   # ← Aggregate 도메인 메서드 호출 → 이벤트 발행
    event_bus.publish(event)

# payment_behavior.py
def handle_order_placed(event: OrderPlacedDomainEvent, event_bus):
    payment = Payment()
    evt2 = payment.complete(event.order_id) # ← Aggregate 도메인 메서드 호출 → 이벤트 발행
    event_bus.publish(evt2)

# shipping_behavior.py
def handle_payment_completed(event: PaymentCompletedDomainEvent, mq_publisher):
    shipping = Shipping()
    evt3 = shipping.initiate(event.order_id)
    mq_publisher.publish(evt3)              # ← Adapter 통해 MQ 송신
```

---

## 마무리

도메인 이벤트는 마이크로서비스 아키텍처에서 "트랜잭션 경계를 넘어 어그리게이트를 연결"하는 핵심 도구입니다.

- **직접 의존성을 낮추고 느슨한 결합**을 유지하며,
- **비즈니스 규칙 중심 설계**를 가능하게 합니다.

다만 이 패턴은 **단건 트랜잭션**에서는 매우 깔끔하지만, **여러 어그리게이트를 한 번에 처리**해야 하는 경우에는 한계가 드러납니다. **2편**에서는 이 한계를 실제로 만났던 사례를 바탕으로, **도메인 서비스(Domain Service)**를 사용해 어떻게 보완했는지 공유하겠습니다.
