---
title: "nats-py 실전 삽질기 — silent zombie부터 DLQ까지"
date: 2026-03-13
categories: ["infra"]
tags: ["NATS", "JetStream", "nats-py", "Python", "DLQ", "Dedup", "ECS", "헬스체크", "asyncio"]
series: ["Python 마이크로서비스에 NATS JetStream 도입기"]
series_order: 3
project: "backend"
source_sessions: []
---

NATS 인프라를 올리고 이벤트 파이프라인을 설계했습니다. 이제 Python으로 Consumer를 구현하면 됩니다. nats-py 라이브러리를 쓰면 간단해 보이지만, 프로덕션에서 안정적으로 돌리려면 몇 가지 함정이 있습니다.

## 🧟 Silent Zombie — 연결은 살아있는데 메시지가 안 온다

가장 먼저 부딪힌 문제입니다.

**증상**: NATS 서버가 재시작된 뒤 Consumer 프로세스가 살아있는데 메시지를 수신하지 않습니다. 로그에 에러도 없습니다. CPU도 0%입니다. 프로세스는 정상인데 아무 일도 일어나지 않습니다.

**원인**: nats-py는 TCP 연결이 끊기면 자동으로 reconnect합니다. TCP 레벨에서는 연결이 복구됩니다. 그런데 **JetStream push subscription은 복원되지 않습니다.** NATS 서버 입장에서는 새 연결이 맺어졌지만, 이전에 등록했던 push subscription 정보가 없습니다. Consumer는 연결이 살아있으니 정상이라고 판단하고, 서버는 구독이 없으니 메시지를 보내지 않습니다.

이 상태가 **silent zombie**입니다. 에러 없이 조용히 죽어있습니다.

**해결**: reconnect 콜백에서 즉시 프로세스를 종료합니다.

```python
async def reconnected_cb():
    logger.critical(
        "NATS reconnected — JetStream subscriptions NOT restored. "
        "Exiting to let ECS restart the task."
    )
    sys.exit(1)

nc = await nats.connect(
    nats_url,
    reconnected_cb=reconnected_cb,
)
```

프로세스가 종료되면 ECS가 태스크를 재시작합니다. 새 프로세스가 뜨면서 JetStream subscription을 다시 등록합니다. 우아하지는 않지만, nats-py가 JetStream subscription 복원을 지원할 때까지는 이 방법이 가장 확실합니다.

nats-py의 기본 reconnect 동작이 JetStream 사용자에게는 함정입니다. TCP reconnect 성공 = 정상 복구라고 착각하기 쉽습니다. 공식 문서에도 이 제약이 명시적으로 안내되어 있지 않았습니다.

## 🤫 Consumer Config Silent Drop

두 번째 함정입니다.

**증상**: Consumer의 `max_deliver` 값을 5로 변경했는데, 실제로는 이전 값인 -1(무제한)로 동작합니다.

**원인**: nats-py에서 `js.subscribe()`를 호출할 때 `ConsumerConfig`를 전달할 수 있습니다. 그런데 **같은 이름의 durable consumer가 이미 존재하면, 새로 전달한 config를 조용히 무시합니다.** 에러도 안 나고, 경고도 안 나옵니다. 기존 consumer의 설정이 그대로 유지됩니다.

```python
# max_deliver=5로 설정했지만...
await js.subscribe(
    "billing.>",
    queue="ef-billing",
    stream="BILLING",
    config=ConsumerConfig(max_deliver=5),
)
# 이미 ef-billing consumer가 max_deliver=-1로 존재하면
# 5가 아니라 -1로 동작합니다. 에러 없음!
```

**해결**: subscribe 후에 `consumer_info()`로 실제 설정을 검증합니다.

```python
sub = await js.subscribe(
    "billing.>",
    queue="ef-billing",
    stream="BILLING",
    config=ConsumerConfig(max_deliver=5),
)

# 실제 consumer 설정 검증
info = await sub.consumer_info()
actual = info.config.max_deliver
expected = 5

if actual != expected:
    logger.critical(
        f"Consumer config mismatch! "
        f"Expected max_deliver={expected}, got {actual}. "
        f"Delete consumer and restart."
    )
```

config를 변경하려면 기존 consumer를 삭제하고 재생성해야 합니다. `nats consumer del BILLING ef-billing` 후 서비스를 재시작하면 됩니다.

## 🏥 DLQ — 반복 실패 메시지 격리

메시지 처리(forwarding)가 실패하면 `msg.nak()`으로 NATS에 재전달을 요청합니다. 그런데 특정 메시지가 계속 실패하면 어떻게 될까요? 무한 재시도는 다른 메시지 처리를 지연시킵니다.

앱 레벨 DLQ(Dead Letter Queue)를 구현했습니다.

```
forwarding 실패 (attempt < 3) → msg.nak() → NATS 재전달
forwarding 실패 (attempt ≥ 3) → DLQ publish → msg.ack()
DLQ publish 실패            → msg.nak() → NATS max_deliver=5 여유
```

3회 실패한 메시지는 별도 DLQ Stream으로 보내고 원본은 ack 처리합니다. DLQ에 보관된 메시지는 나중에 원인을 파악한 뒤 수동으로 replay할 수 있습니다.

### DLQ 메시지 구조

```json
{
  "original_subject": "billing.subscription.activated",
  "original_event": { "...CloudEvent..." },
  "error": {
    "message": "Connection refused",
    "type": "ConnectionError",
    "attempts": 3
  },
  "failed_at": "2026-03-12T09:00:00+00:00",
  "source_stream": "BILLING",
  "consumer": "ef-billing"
}
```

원본 이벤트, 에러 정보, 실패 시각, 출처를 모두 담습니다. 운영 시 이 정보만으로 어떤 이벤트가 왜 실패했는지 파악할 수 있습니다.

### 왜 앱 레벨인가

NATS JetStream 자체에도 `max_deliver` 설정이 있습니다. `max_deliver=5`로 설정하면 5회 전달 후 NATS가 메시지를 버립니다. 하지만 **버려진 메시지가 어디에도 남지 않습니다.** 왜 실패했는지, 어떤 이벤트였는지 추적이 안 됩니다.

앱 레벨 DLQ는 NATS의 `max_deliver`보다 먼저 동작합니다. 앱이 3회째 실패를 감지하면 DLQ로 보내고 ack합니다. NATS의 `max_deliver=5`는 DLQ publish 자체가 실패하는 극단적인 경우를 위한 안전망입니다.

| 계층 | 임계치 | 역할 |
|------|--------|------|
| 앱 DLQ | 3회 | 실패 메시지를 DLQ Stream으로 격리 + 메타데이터 보존 |
| NATS max_deliver | 5회 | DLQ 전송도 실패할 때의 최후 안전망 |

## 🔄 Dedup — NATS KV로 중복 방지

NATS JetStream은 at-least-once를 보장합니다. 즉 같은 메시지가 두 번 이상 전달될 수 있습니다. Consumer 재생성(deliver policy: all)이나 네트워크 타이밍 이슈로 중복이 발생합니다.

NATS KV(Key-Value) bucket으로 dedup을 구현했습니다.

```python
kv = await js.key_value(bucket="PROCESSED_EVENTS")

async def handle(msg):
    envelope = orjson.loads(msg.data)
    event_id = envelope["id"]  # CloudEvents id

    # 이미 처리된 이벤트인지 확인
    try:
        await kv.get(event_id)
        # 키가 존재하면 중복 → skip
        await msg.ack()
        return
    except KeyNotFoundError:
        pass

    # 이벤트 처리
    await forwarder.forward(envelope)

    # 처리 완료 기록
    await kv.put(event_id, b"1")
    await msg.ack()
```

| 설정 | 값 |
|------|-----|
| Bucket | `PROCESSED_EVENTS` |
| TTL | 30일 |
| Key | CloudEvents `id` 필드 |

TTL 30일이면 30일 전 이벤트가 재전달되지 않는 한 중복이 방지됩니다. KV bucket은 JetStream 위에 구현되어 있어서 별도 Redis 같은 외부 저장소가 필요 없습니다.

CloudEvents 스펙의 `id` 필드를 dedup key로 바로 쓸 수 있다는 점이 CloudEvents를 선택한 이유 중 하나였습니다.

## 📋 에러 처리 분류

모든 에러를 같은 방식으로 처리하면 안 됩니다. 에러 성격에 따라 세 가지로 분류했습니다.

| 에러 유형 | 처리 | 이유 |
|-----------|------|------|
| 파싱 실패 (non-JSON, 빈 메시지) | `msg.term()` | 재전달해도 같은 실패 반복 |
| Forwarding 실패 (일시적) | `msg.nak()` → 재전달 | 네트워크 장애 등, 재시도로 해결 가능 |
| Forwarding 3회 실패 | DLQ → `msg.ack()` | 반복 실패 메시지 격리 |

`msg.term()`은 NATS에 "이 메시지는 다시 보내지 마라"고 알리는 것입니다. 깨진 JSON을 아무리 재전달해도 파싱은 성공하지 않습니다. `msg.nak()`은 "나중에 다시 보내달라"입니다. `msg.ack()`은 "처리 완료"입니다.

## ✅ 파일 기반 헬스체크 — HTTP 없는 워커의 생존 보고

Consumer 워커는 HTTP 엔드포인트가 없습니다. FastAPI를 안 씁니다. 순수 asyncio로 NATS를 구독만 합니다. 그런데 ECS는 태스크가 살아있는지 확인해야 합니다.

파일 기반 헬스체크를 구현했습니다.

```python
async def health_check_loop(nc: Client):
    while True:
        if nc.is_connected:
            Path("/tmp/healthy").touch()
        await asyncio.sleep(15)
```

15초마다 NATS 연결 상태를 확인하고, 연결이 살아있으면 `/tmp/healthy` 파일을 touch합니다. ECS Task Definition의 헬스체크는 이 파일이 최근 1분 내에 수정됐는지 확인합니다.

```bash
# ECS Task Definition healthCheck
CMD-SHELL: find /tmp/healthy -mmin -1 | grep -q . || exit 1
# interval: 30s, timeout: 5s, retries: 3, startPeriod: 60s
```

파일이 없거나 1분 이상 오래되면 unhealthy → ECS가 태스크를 재시작합니다. `startPeriod: 60s`로 초기 구동 시간을 허용합니다.

이 패턴은 HTTP 서버를 띄우지 않는 워커 컨테이너에서 범용적으로 쓸 수 있습니다. sidecar로 헬스체크 서버를 붙이는 것보다 단순합니다.

## 🏁 전체 메시지 처리 흐름

지금까지 설명한 내용을 하나의 흐름으로 정리합니다.

```
NATS JetStream
  │
  ▼
handle(msg)
  ├─ orjson.loads() 실패 → msg.term()  (파싱 불가, 재전달 무의미)
  ├─ Dedup 체크 (NATS KV: PROCESSED_EVENTS)
  │   └─ 이미 처리된 event_id → skip + msg.ack()
  ├─ forwarder.forward(cloud_event)
  │   ├─ 성공 → KV에 event_id 기록 → msg.ack()
  │   └─ 실패 (attempt < 3) → msg.nak()  (JetStream이 재전달)
  └─ 실패 (attempt ≥ 3) → DLQ publish → msg.ack()
      └─ DLQ publish 실패 → msg.nak()  (NATS max_deliver=5 여유)
```

메시지 하나가 들어오면: 파싱 → dedup → forwarding → 성공/실패 분기. 각 단계에서 적절한 ack/nak/term을 호출합니다. DLQ와 NATS max_deliver가 이중 안전망을 형성합니다.

## 📌 nats-py 사용 시 체크리스트

프로덕션에서 nats-py를 쓴다면 확인해야 할 항목을 정리합니다.

| 항목 | 확인 |
|------|------|
| reconnected_cb에서 exit 처리 | JetStream push subscription은 복원 안 됨 |
| subscribe 후 consumer_info() 검증 | config가 silent drop될 수 있음 |
| queue 파라미터만 사용 (durable과 별도 지정 금지) | nats-py 제약, 에러 발생 |
| msg.term() / nak() / ack() 분류 | 에러 성격에 따라 다르게 처리 |
| 앱 레벨 DLQ | NATS max_deliver만으로는 메타데이터 유실 |
| Dedup (KV 또는 외부 저장소) | at-least-once 보장 = 중복 가능 |
| HTTP 없는 워커의 헬스체크 | 파일 기반 또는 별도 방법 필요 |

3편에 걸쳐 NATS JetStream 인프라 구축, 이벤트 파이프라인 설계, Consumer 안정성 확보를 다뤘습니다. nats-py 라이브러리가 성숙해지면 reconnect 문제 같은 워크어라운드는 불필요해질 수 있지만, DLQ/Dedup/에러 분류 같은 패턴은 어떤 메시지 브로커를 쓰든 적용됩니다.
