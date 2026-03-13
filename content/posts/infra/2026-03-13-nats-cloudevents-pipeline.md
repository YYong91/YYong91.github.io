---
title: "CloudEvents 기반 이벤트 파이프라인 — subject 컨벤션부터 DDD 발행까지"
date: 2026-03-13
categories: ["infra"]
tags: ["NATS", "JetStream", "CloudEvents", "이벤트", "DDD", "마이크로서비스", "Python", "TypedDict"]
series: ["Python 마이크로서비스에 NATS JetStream 도입기"]
series_order: 2
project: "backend"
source_sessions: []
---

NATS 인프라를 올렸으면 그 위에 이벤트를 흘려야 합니다. 메시지 포맷, subject 네이밍, 스키마 관리, 발행 패턴을 정했습니다.

## 🏁 설계 결정 3가지

이벤트 파이프라인을 설계하면서 먼저 세 가지를 결정했습니다.

1. **메시지 포맷**: CloudEvents 1.0
2. **Subject 네이밍**: `{service}.{entity}.{event}`
3. **스키마 관리**: 공유 패키지로 Single Source of Truth

각각의 이유를 설명합니다.

## 📦 CloudEvents 1.0 — 왜 자체 포맷이 아닌가

이벤트 메시지를 어떤 구조로 보낼지 정해야 합니다. 자체 포맷을 만들 수도 있지만, CloudEvents 1.0 스펙을 채택했습니다.

```json
{
  "specversion": "1.0",
  "id": "01JNXXXXXXXXXXXXXX",
  "source": "billing",
  "type": "billing.subscription.activated",
  "time": "2026-03-10T09:00:00+09:00",
  "datacontenttype": "application/json",
  "data": {
    "event_id": "01JNXXXXXXXXXXXXXX",
    "aggregate_id": 123456789,
    "customer_id": 987654321,
    "plan_id": 111222333
  }
}
```

| 필드 | 역할 |
|------|------|
| `specversion` | 항상 `"1.0"` — 파서가 버전을 판단 |
| `id` | TSID 기반 이벤트 고유 ID — 중복 방지에 사용 |
| `source` | 발행 서비스명 — 누가 보냈는지 |
| `type` | NATS subject와 동일 — 어떤 이벤트인지 |
| `time` | ISO 8601, timezone-aware — 언제 발생했는지 |
| `data` | 이벤트별 실제 페이로드 |

CloudEvents를 쓰면 좋은 점은 세 가지입니다.

**첫째, envelope 구조를 논의할 필요가 없습니다.** `id`는 어디에, `timestamp`는 무슨 포맷으로 — 이런 결정을 스펙이 대신 해줍니다. 새 서비스가 이벤트를 발행할 때 "메시지 구조 어떻게 하지?"가 아니라 "data에 뭘 넣지?"만 고민하면 됩니다.

**둘째, 외부 시스템과 호환됩니다.** RudderStack, Segment 같은 분석 도구가 CloudEvents를 지원합니다. Consumer 워커가 외부로 전달할 때 포맷 변환 없이 그대로 보낼 수 있습니다.

**셋째, `id` 필드가 dedup에 직접 쓰입니다.** Consumer 쪽에서 NATS KV에 `id`를 기록해 중복을 방지합니다. 별도 dedup key를 설계할 필요가 없습니다.

## 🏷️ Subject 네이밍 — 3-depth 계층

```
{service}.{entity}.{event}
```

| Subject | 의미 |
|---------|------|
| `billing.subscription.activated` | 결제 서비스의 구독 활성화 |
| `billing.credit.reserved` | 결제 서비스의 크레딧 예약 |
| `account.user.signed-up` | 인증 서비스의 사용자 가입 |

이 패턴을 고른 이유는 NATS의 wildcard 구독과 맞물리기 때문입니다.

| 패턴 | 구독 범위 |
|------|---------|
| `billing.>` | 결제 서비스 전체 이벤트 |
| `billing.subscription.>` | 구독 관련 이벤트만 |
| `billing.subscription.activated` | 특정 이벤트 하나 |

서비스별로 독립 Stream을 만들어 `{service}.>`로 바인딩합니다. Consumer는 전체를 구독할 수도, 특정 entity만 필터링할 수도 있습니다. event 이름은 kebab-case로 통일했습니다 (`signed-up`, `trial-started`).

## 📐 이벤트 스키마 패키지 — 계약의 Single Source of Truth

Subject 문자열을 서비스마다 하드코딩하면 오타가 나옵니다. `"billing.subscripton.activated"` 같은 실수는 런타임에서야 발견됩니다.

이를 방지하기 위해 이벤트 스키마 전용 공유 패키지를 만들었습니다. 두 가지를 제공합니다.

### Subject 상수

```python
# events_schema/billing/subjects.py

CREDIT_RESERVED = "billing.credit.reserved"
CREDIT_LOCKED = "billing.credit.locked"
CREDIT_CONSUMED = "billing.credit.consumed"
CREDIT_RELEASED = "billing.credit.released"

SUBSCRIPTION_ACTIVATED = "billing.subscription.activated"
SUBSCRIPTION_RENEWED = "billing.subscription.renewed"
# ...
```

Producer와 Consumer가 같은 상수를 import합니다. 오타가 나면 import 에러로 빌드 시점에 잡힙니다.

### Payload TypedDict

```python
# events_schema/billing/subscription.py

class SubscriptionActivatedPayload(TypedDict):
    event_id: str
    event_type: str
    occurred_at: str
    aggregate_id: int
    customer_id: int
    plan_id: int
    gateway_subscription_id: str
```

Consumer가 이벤트를 파싱할 때 어떤 필드가 오는지 코드에서 바로 확인할 수 있습니다. IDE 자동완성도 됩니다.

```python
from events_schema.billing.subjects import SUBSCRIPTION_ACTIVATED
from events_schema.billing.subscription import SubscriptionActivatedPayload

async def handler(msg):
    envelope = orjson.loads(msg.data)
    if msg.subject == SUBSCRIPTION_ACTIVATED:
        data: SubscriptionActivatedPayload = envelope["data"]
        customer_id = data["customer_id"]  # IDE 자동완성 ✓
```

### 패키지 구조

```
events_schema/
├── billing/
│   ├── subjects.py          # subject 상수
│   ├── credit.py            # credit 관련 payload TypedDict
│   └── subscription.py      # subscription 관련 payload TypedDict
├── account/
│   ├── subjects.py
│   ├── user.py
│   └── organization.py
└── llmgateway/
    ├── subjects.py
    └── ...
```

서비스별 디렉토리가 분리되어 있고, 새 이벤트를 추가할 때 `subjects.py`에 상수 + payload TypedDict를 함께 추가합니다. 이 패키지가 이벤트 계약의 유일한 진실 공급원입니다.

## 🔧 발행 패턴 — DDD vs Non-DDD

서비스마다 아키텍처가 다릅니다. 결제 서비스는 DDD로 구현되어 있고, 인증 서비스는 전통적인 레이어드 아키텍처입니다. 이벤트 발행 방식도 달라야 했습니다.

### DDD 패턴

DDD 서비스는 도메인 레이어가 NATS의 존재를 모릅니다. Aggregate가 상태를 변경하면 도메인 이벤트를 내부에 등록하고, Infrastructure 레이어가 이를 수집해서 발행합니다.

```
Domain Layer                    Infrastructure Layer
┌─────────────────────┐        ┌──────────────────────────────┐
│ Aggregate           │        │ Repository.save()            │
│ └─ register_event() │───────▶│ └─ EventCollector.collect()  │
│                     │        │                              │
│ (NATS 모름)          │        │ UoW.save_changes()           │
│                     │        │ ├─ collector.drain()         │
└─────────────────────┘        │ ├─ session.commit()          │
                               │ └─ publisher.publish()       │
                               │    └─ CloudEvents → JetStream│
                               └──────────────────────────────┘
```

**EventCollector 패턴**이 핵심입니다. 하나의 Unit of Work 안에서 여러 Aggregate가 이벤트를 등록하면, EventCollector가 이를 모았다가 `save_changes()` 시점에 일괄 발행합니다.

```python
# 도메인: NATS를 모르는 순수 Python
class Subscription(AggregateRoot):
    def activate(self, gateway_subscription_id: str, ...):
        self._status = SubscriptionStatus.ACTIVE
        self.register_event(
            SubscriptionActivated(
                aggregate_id=self.id,
                customer_id=self.customer_id,
                gateway_subscription_id=gateway_subscription_id,
            )
        )
```

```python
# 인프라: DB commit 후 best-effort 발행
async def save_changes(self) -> None:
    events = self._collector.drain()
    await self._session.commit()
    try:
        await self._publisher.publish(events)
    except Exception:
        logger.error("Event publish failed", exc_info=True)
        # DB commit은 성공. 이벤트 실패는 로그만 남깁니다.
```

DB commit이 성공한 뒤 이벤트를 발행합니다. 이벤트 발행이 실패해도 DB 트랜잭션은 롤백하지 않습니다. 이것이 **best-effort 발행** 전략입니다. 이벤트 유실 가능성은 있지만, 트랜잭션과 이벤트 발행을 하나의 원자적 연산으로 묶으려면 Outbox 패턴이 필요하고, 현 단계에서는 복잡도 대비 실익이 적다고 판단했습니다.

Domain Event 클래스와 NATS Subject의 매핑은 Event Registry가 담당합니다.

```python
# infrastructure/events/registry.py
EVENT_SUBJECT_MAP: dict[type, str] = {
    SubscriptionActivated: SUBSCRIPTION_ACTIVATED,
    CreditReserved: CREDIT_RESERVED,
    # ...
}
```

### Non-DDD 패턴

DDD를 적용하지 않는 서비스는 더 단순합니다. `AppEvent`를 직접 생성해서 발행합니다.

```python
from shared.events import AppEvent
from events_schema.account.subjects import USER_SIGNED_UP

event = AppEvent(
    event_type=USER_SIGNED_UP,
    data={"user_id": 123, "email": "user@example.com"},
)
await app.state.event_publisher.publish([event])
```

`event_type`이 NATS subject가 됩니다. 같은 `EventPublisher` 인터페이스를 쓰지만, 도메인 이벤트 → subject 변환 없이 직접 subject를 지정합니다.

### 공통 인터페이스

두 패턴 모두 공유 패키지의 `EventPublisher` 추상 인터페이스를 사용합니다.

```python
from shared.events import EventPublisher

await publisher.publish([event1, event2])
```

구현체는 `NatsEventPublisher`(실제 발행)와 `FakeEventPublisher`(테스트)가 있습니다. 서비스 코드는 구현체를 모르고, 테스트에서는 Fake를 주입해서 실제 NATS 없이 이벤트 발행을 검증합니다.

```python
async def test_subscription_activated_publishes_event():
    # Given
    publisher = FakeEventPublisher()
    uow = FakeUnitOfWork(event_publisher=publisher)

    # When
    async with uow:
        subscription.activate(gateway_subscription_id="sub_xxx")
        await uow.subscriptions.save(subscription)
        await uow.save_changes()

    # Then
    assert len(publisher.published) == 1
    assert isinstance(publisher.published[0], SubscriptionActivated)
```

## 📊 새 이벤트 추가 절차

정리하면, 새 이벤트를 추가할 때 거치는 단계입니다.

| 단계 | 파일 | 내용 |
|------|------|------|
| 1 | `events-schema/{svc}/subjects.py` | Subject 상수 추가 |
| 2 | `events-schema/{svc}/{entity}.py` | Payload TypedDict 추가 |
| 3-A (DDD) | `domain/events/` + `registry.py` | DomainEvent 클래스 + Registry 매핑 |
| 3-B (Non-DDD) | 로직 레이어 | `AppEvent(event_type=..., data=...)` |
| 4 | 테스트 | `FakeEventPublisher`로 발행 검증 |

스키마 패키지 변경 → 발행 서비스 코드 → 테스트 순서입니다. Consumer 쪽은 스키마 패키지의 상수와 TypedDict를 import해서 파싱하면 됩니다.

다음 글에서는 Consumer 측 안정성 — nats-py의 reconnect 문제, DLQ, Dedup, 파일 기반 헬스체크 — 을 다룹니다.
