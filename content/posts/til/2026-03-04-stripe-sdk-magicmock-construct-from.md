---
title: "MagicMock으로 Stripe SDK를 테스트하면 안 되는 이유"
date: 2026-03-04
categories: ["til"]
tags: ["Stripe", "pytest", "Python", "testing"]
project: "CUBIG Backend"
source_sessions: ["2026-03-04-stripe-test-fixes"]
summary: "StripeObject는 dict 서브클래스다. MagicMock은 이 특성을 숨긴다."
---

billing 서비스에서 Stripe adapter 테스트를 작성하다가 이상한 현상을 만났다. 로컬에서는 테스트가 통과하는데, Stripe API 응답 구조가 바뀐 후 실제 sandbox에서만 오류가 나는 케이스였다.

## 문제: MagicMock은 StripeObject를 제대로 흉내내지 못한다

기존 코드는 이랬다.

```python
# 테스트에서
mock_sub = MagicMock()
mock_sub.id = "sub_test_123"
mock_sub.status = "active"
mock_sub.items = MagicMock()
mock_sub.items.data = [MagicMock()]
```

겉으로 보면 문제없어 보인다. 하지만 Stripe SDK의 `StripeObject`는 `dict`의 서브클래스다.

```python
# stripe SDK 내부
class StripeObject(dict):
    ...
```

`StripeObject`를 순회(`for key, value in obj.items()`)하면 Python의 `dict.items()`가 호출된다. 반면 `MagicMock`에서 `mock_sub.items`는 새로운 `MagicMock`을 반환하고, `dict.items()`처럼 동작하지 않는다. 실제 Stripe SDK 내부에서 `.items` attribute에 접근하는 방식과 테스트에서의 접근 방식이 다르다는 얘기다.

```python
# 실제 Stripe SDK 내부 동작 (단순화)
subscription = stripe_api_response  # StripeObject (dict subclass)

# items는 dict.items()가 아니라 subscription["items"] (StripeObject)에 접근
period_start = subscription["items"]["data"][0]["current_period_start"]
```

MagicMock으로는 이 `.items` vs `dict.items()` 충돌을 재현할 수 없다. 테스트가 통과해도 실제 SDK 동작과 다르다.

## 해결: construct_from으로 실제 StripeObject 생성

`stripe.Subscription.construct_from()`을 사용하면 실제 Stripe SDK 객체를 직접 생성할 수 있다.

```python
mock_sub = stripe.Subscription.construct_from(
    {
        "id": "sub_test_123",
        "object": "subscription",
        "status": "active",
        "items": {
            "object": "list",
            "data": [
                {
                    "id": "si_456",
                    # Stripe API 2025+: current_period는 item 레벨에 위치
                    "current_period_start": 1700000000,
                    "current_period_end": 1702592000,
                    "price": {
                        "id": "price_test_789",
                        "product": "prod_abc",
                    },
                }
            ],
        },
        "cancel_at_period_end": False,
        "metadata": {},
    },
    key="sk_test_dummy",
)
```

이렇게 하면 dict 서브클래스로서의 특성이 그대로 유지된다. 실제 sandbox API 없이도 SDK의 내부 동작을 그대로 테스트할 수 있다.

## Stripe API 2025+ 구조 변경: current_period 위치 이동

이번에 함께 발견한 건데, Stripe API 버전이 올라가면서 `current_period_start`, `current_period_end`의 위치가 바뀌었다.

| Stripe API | current_period 위치 |
|------------|---------------------|
| 2024 이전  | `subscription` 최상위 레벨 |
| 2025+      | `subscription.items.data[0]` (item 레벨) |

기존 코드가 `subscription.current_period_start`를 직접 읽고 있었다면, 이제 `subscription["items"]["data"][0]["current_period_start"]`를 읽어야 한다. `construct_from`으로 테스트 픽스처를 제대로 구성하지 않으면 이 변경을 잡아내지 못한다.

## fixture 환경변수 이름도 맞춰야 한다

작은 함정이 하나 더 있었다. `conftest.py`에서 환경변수 이름이 실제 코드와 달랐다.

```python
# conftest.py — 픽스처가 설정한 이름
os.environ["STRIPE_API_KEY"] = "sk_test_xxx"

# 실제 코드에서 읽는 이름
stripe_key = os.environ["STRIPE_SECRET_KEY"]  # KeyError 발생
```

`STRIPE_API_KEY`와 `STRIPE_SECRET_KEY`는 다르다. Stripe 공식 문서의 관행은 `STRIPE_SECRET_KEY`를 사용한다. 환경변수 이름을 통일하고 나서 테스트가 정상 실행됐다.

## 요약

- Stripe SDK 테스트에는 `MagicMock` 대신 `construct_from()` 사용
- Stripe API 2025+에서 `current_period_*`는 subscription 최상위가 아닌 item 레벨에 있음
- 픽스처 환경변수명과 실제 코드의 환경변수명이 일치하는지 확인할 것
