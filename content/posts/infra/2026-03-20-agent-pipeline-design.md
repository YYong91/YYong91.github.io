---
title: "AI Agent 실행 파이프라인 설계기 — NATS, Temporal, AWS Batch로 만든 비동기 오케스트레이션"
date: 2026-03-20
categories: ["infra"]
tags: ["NATS", "Temporal", "AWS Batch", "비동기", "오케스트레이션", "파이프라인", "Python", "마이크로서비스", "EventBridge", "Lambda"]
series: ["Python 마이크로서비스에 NATS JetStream 도입기"]
series_order: 4
project: "backend"
source_sessions: []
---

AI Agent를 실행하는 데 걸리는 시간은 30초일 수도, 2시간일 수도 있습니다. 이런 작업을 HTTP 요청 하나로 처리할 수는 없습니다.

## 🎯 풀어야 할 문제

Agent 서비스가 받은 실행 요청을 어딘가에서 실제로 수행하고, 결과를 돌려줘야 합니다. 문제는 이 "어딘가"의 특성입니다.

| 특성 | 설명 |
|------|------|
| 실행 시간 | 수십 초 ~ 수 시간 (예측 불가) |
| 컴퓨팅 파워 | GPU 또는 고사양 CPU 필요 |
| 실패 가능성 | OOM, timeout, 외부 API 에러 |
| 취소 요구 | 사용자가 실행 중 취소할 수 있어야 함 |
| 상태 추적 | 시작/완료/실패를 실시간으로 알아야 함 |

단순히 컨테이너를 띄우고 끝나면 결과를 가져오는 것으로는 부족합니다. 실행을 "관리"해야 합니다. 상태를 추적하고, 실패하면 기록하고, 취소 요청이 들어오면 중단하고, 이 모든 이벤트를 다른 서비스에 전파해야 합니다.

## 🏗️ 전체 아키텍처

10단계로 나뉘는 전체 흐름입니다.

![Agent Pipeline Architecture](/images/agent-pipeline-architecture.png)

```text
Client
  │
  ▼
Agent (ECS)
  │  ① 실행 요청 접수 + S3에 입력 데이터 업로드
  │  ② NATS에 agent.run.requested 발행
  │
  ▼
Worker Runner (ECS)
  │  ③ NATS subscribe (push + queue group)
  │  ④ Temporal workflow 시작
  │  ⑤ AWS Batch job 제출 (task_token 전달)
  │
  ▼
AWS Batch
  │  ⑥ AI Agent 컨테이너 실행 (장시간)
  │  ⑦ 완료/실패 시 EventBridge 이벤트 발생
  │
  ▼
EventBridge → Lambda
  │  ⑧ task_token으로 Temporal activity async complete
  │
  ▼
Worker Runner (Temporal workflow 재개)
  │  ⑨ 최종 상태 업데이트 (gRPC → Agent 서비스)
  │  ⑩ 이벤트 발행 (NATS CloudEvents)
```

핵심은 **Worker Runner**입니다. NATS에서 메시지를 받아 Temporal workflow를 시작하고, Temporal이 Batch 제출부터 결과 처리까지 전체 흐름을 오케스트레이션합니다.

## ⚖️ 핵심 설계 결정 3가지

### 왜 NATS인가

Agent 서비스와 Worker Runner 사이의 메시지 전달에 NATS JetStream을 사용합니다.

| 후보 | 탈락 이유 |
|------|----------|
| SQS | AWS 종속. 온프레미스 납품 시 대체재 필요 |
| RabbitMQ | 운영 복잡도 높음, Erlang 클러스터 관리 |
| Kafka | 이 규모에서 과도한 인프라 |
| **NATS** | **클라우드/온프레미스 동일 코드, 경량, JetStream으로 at-least-once 보장** |

온프레미스 환경에도 납품될 수 있는 서비스입니다. NATS는 단일 바이너리로 설치할 수 있고, JetStream이 persistence를 제공합니다. AWS에서는 ECS로, 온프레미스에서는 K3s로 — 코드 변경 없이 동일하게 동작합니다.

### 왜 Temporal인가

"Batch job 제출하고 결과 받으면 상태 업데이트" — 이 로직을 코드로 직접 관리하면 어떻게 될까요?

```python
# 이렇게 하면 안 됩니다
async def handle_run(msg):
    await update_status(RUNNING)           # 여기서 죽으면?
    job_id = await submit_batch(msg)       # 제출 후 죽으면?
    result = await wait_for_result(job_id) # 2시간 동안 메모리에?
    await update_status(COMPLETED)         # 여기서 실패하면?
```

프로세스가 중간에 죽으면 어디까지 진행했는지 알 수 없습니다. 재시작하면 처음부터 다시 해야 합니다. Batch job은 이미 제출됐는데 또 제출할 수도 있습니다.

Temporal은 이 문제를 해결합니다.

| Temporal이 주는 것 | 설명 |
|--------------------|------|
| Durable execution | 프로세스가 죽어도 workflow는 마지막 activity 이후부터 재개 |
| Visibility | Temporal UI에서 실행 중인 workflow, 히스토리, 에러를 실시간 확인 |
| Signal | 외부에서 실행 중인 workflow에 메시지 전달 (취소, job_id 설정) |
| Timeout | activity 단위로 timeout 설정, 초과 시 자동 실패 처리 |
| Retry | activity 단위 retry policy (exponential backoff) |

직접 구현하면 이 중 하나도 제대로 만들기 어렵습니다. 다섯 개를 동시에 얻을 수 있어서 Temporal을 선택했습니다.

### 왜 AWS Batch인가

AI Agent는 실행 시간이 수 분에서 수 시간, 필요한 리소스도 작업마다 다릅니다. ECS Task를 직접 띄우는 것보다 Batch가 나은 이유가 있습니다.

| 비교 항목 | ECS Task (직접) | AWS Batch |
|-----------|----------------|-----------|
| 리소스 할당 | 고정 task definition | job별 CPU/메모리 동적 지정 |
| 큐잉 | 직접 구현 | 내장 job queue + priority |
| 스케일링 | ASG 직접 관리 | Compute Environment가 자동 확장 |
| EventBridge 연동 | 별도 설정 | job 상태 변경 이벤트 자동 발생 |

`containerOverrides`로 job마다 다른 리소스를 지정할 수 있다는 점이 결정적이었습니다.

```python
response = batch_client.submit_job(
    jobName=f"agent-run-{input.profile_name}-{input.run_id}",
    jobQueue=_settings.batch_job_queue,
    jobDefinition=input.job_definition,
    containerOverrides={
        "vcpus": input.resource.get("cpu", 1),
        "memory": input.resource.get("memory", 4096),
        "environment": env_vars,
    },
)
```

## 🔄 Temporal Async Completion — 가장 까다로운 부분

일반적인 Temporal activity는 함수가 return하면 완료됩니다. 하지만 Batch job은 수 시간 걸릴 수 있습니다. activity 함수 안에서 polling으로 기다리면 그 동안 worker 스레드를 점유합니다.

**Async Completion** 패턴은 이 문제를 해결합니다.

```text
Worker Runner                    AWS Batch                   Lambda
    │                               │                          │
    ├─ submit_job()                 │                          │
    │   task_token을 env var로 전달  │                          │
    │                               │                          │
    ├─ raise_complete_async() ──┐   │                          │
    │   activity를 "대기" 상태로  │   │                          │
    │   worker 스레드 즉시 해제   │   │                          │
    │                            │   │                          │
    │                            │   ├── (수 시간 실행)         │
    │                            │   │                          │
    │                            │   ├── 완료 → EventBridge     │
    │                            │   │                     ──→  │
    │                            │   │        task_token으로     │
    │                            │   │        activity complete  │
    │  ◄─────────────────────────┘───┘──────────────────────────┘
    │   workflow 재개, 결과 처리
```

핵심 코드는 `batch.py`의 마지막 두 줄입니다.

```python
@activity.defn
async def submit_batch_job(input: AgentRunInput) -> BatchResult:
    task_token = activity.info().task_token
    token_b64 = base64.b64encode(task_token).decode()

    # ... Batch job 제출, task_token을 환경변수로 전달 ...

    # activity를 미완료 상태로 두고 즉시 반환
    activity.raise_complete_async()
```

`raise_complete_async()`가 호출되면 activity는 "누군가가 나중에 완료해줄 것"이라는 상태가 됩니다. worker 스레드는 해제되고, Temporal 서버만 이 activity가 아직 완료되지 않았다는 것을 기억합니다.

나중에 Lambda가 `task_token`을 사용해서 activity를 완료합니다. 이때 `BatchResult` dataclass를 전달하면, workflow 입장에서는 마치 activity 함수가 그 값을 return한 것처럼 재개됩니다.

### task_token 인코딩 — E2E 테스트에서 발견한 함정

`task_token`은 Temporal이 내부적으로 사용하는 binary bytes입니다. 이것을 Batch 컨테이너의 환경변수로 전달해야 하므로 문자열로 인코딩해야 합니다.

```python
# Worker Runner (batch.py) — 인코딩
token_b64 = base64.b64encode(task_token).decode()
```

Lambda에서 복원할 때 반드시 `base64.b64decode`를 사용해야 합니다.

```python
# Lambda — 디코딩
task_token = base64.b64decode(os.environ["TEMPORAL_TASK_TOKEN"])
```

E2E 테스트에서 Lambda가 `bytes.fromhex()`로 디코딩하고 있었습니다. base64 문자열을 hex로 디코딩하면 `binascii.Error`가 발생하고, async completion이 실패합니다. Batch job은 성공했는데 workflow가 영원히 대기 상태에 빠지는 증상이었습니다.

에러 자체는 단순하지만, 증상에서 원인까지의 거리가 멉니다. Batch → EventBridge → Lambda → Temporal 순으로 4단계를 거슬러 올라가야 했습니다.

## 🛡️ 안정성 설계

### Cancel 처리 — 2개의 체크포인트

사용자가 실행 중인 Agent를 취소하면 NATS로 `agent.run.cancel-requested` 메시지가 들어옵니다. Temporal workflow에 signal로 전달됩니다.

```python
@workflow.signal
async def cancel(self, input: CancelSignalInput) -> None:
    if self._completed:
        return
    self._cancel_requested = True
    self._cancel_reason = input.reason
```

문제는 cancel signal이 workflow 실행 중 **언제든** 들어올 수 있다는 점입니다. Batch 제출 전에 올 수도 있고, Batch 실행 중에 올 수도 있습니다. 그래서 체크포인트를 2곳에 뒀습니다.

```text
                    Checkpoint 1
                        │
  RUNNING 설정 ──→  cancel 체크  ──→  Batch 제출  ──→  cancel 체크  ──→  최종 상태
                                                          │
                                                    Checkpoint 2
```

| 체크포인트 | 시점 | 동작 |
|-----------|------|------|
| 1 | Batch 제출 전 | 바로 CANCELLED 처리 (Batch 비용 없음) |
| 2 | Batch 결과 수신 후 | `TerminateJob` (best-effort) + CANCELLED 처리 |

Checkpoint 2에서 `cancel_batch_job`은 best-effort입니다. AWS Batch의 `TerminateJob`이 실패해도 workflow는 CANCELLED로 마무리됩니다. 이미 끝난 job을 terminate하려고 해도 문제없습니다.

### 멱등성 — 같은 메시지를 두 번 받아도 안전

NATS JetStream은 at-least-once delivery입니다. 같은 메시지가 두 번 올 수 있습니다.

```python
workflow_id = f"agent-run-{profile_name}-{run_id}"

try:
    await self._temporal.start_workflow(
        workflow_type,
        data,
        id=workflow_id,
        id_conflict_policy=WorkflowIDConflictPolicy.FAIL,
    )
except WorkflowAlreadyStartedError:
    # 이미 시작됨 — 정상 (중복 메시지)
    pass

await msg.ack()
```

Workflow ID를 `agent-run-{profile_name}-{run_id}`로 고정합니다. `run_id`는 TSID이므로 자연스럽게 unique합니다. 동일한 메시지가 두 번 오면 두 번째 시도에서 `WorkflowAlreadyStartedError`가 발생하고, 무시하고 ack합니다.

### Health Check — 연속 실패 임계치

NATS와 Temporal 연결 상태를 15초마다 확인합니다.

```python
HEALTH_FAIL_THRESHOLD = 3  # 연속 3회 실패 시 unhealthy

if nats_ok and temporal_ok:
    consecutive_failures = 0
    HEALTH_FILE.touch()
else:
    consecutive_failures += 1
    if consecutive_failures >= HEALTH_FAIL_THRESHOLD:
        HEALTH_FILE.unlink(missing_ok=True)
```

Temporal의 `check_health()` gRPC 호출은 간헐적으로 실패합니다. 1회 실패로 바로 unhealthy 판정하면 불필요한 컨테이너 재시작이 발생합니다. 연속 3회 실패해야 `/tmp/healthy` 파일을 삭제하고, ECS health check가 이를 감지해서 컨테이너를 교체합니다.

### NATS Reconnect — 재시작이 답

NATS가 reconnect되면 JetStream push subscription은 복원되지 않습니다.

```python
async def _on_reconnected() -> None:
    logger.critical(
        "NATS reconnected — JetStream push "
        "subscriptions are NOT restored. "
        "Exiting for container restart."
    )
    sys.exit(1)
```

복잡한 re-subscribe 로직 대신, `sys.exit(1)`로 프로세스를 종료합니다. ECS가 새 컨테이너를 띄우면 깨끗한 상태에서 subscription을 다시 만듭니다.

## 📐 현재 상태와 다음 단계

Worker Runner 서비스의 현재 구성입니다.

| 구성 요소 | 파일 | 역할 |
|-----------|------|------|
| `subscriber.py` | NATS push subscriber | 메시지 수신 → workflow 시작 |
| `agent_workflow.py` | Temporal workflow | 전체 흐름 오케스트레이션 |
| `batch.py` | Activity | AWS Batch 제출 + async completion |
| `status.py` | Activity | gRPC로 상태 업데이트 |
| `cancel.py` | Activity | Batch job terminate |
| `event.py` | Activity | NATS CloudEvents 발행 |

아직 남은 과제가 있습니다.

- **모니터링**: Temporal workflow 지표를 Prometheus/Grafana로 수집
- **DLQ 처리**: NATS `max_deliver=5` 초과 메시지 처리 정책
- **다중 workflow 타입**: Agent 종류별 다른 workflow 지원 (현재는 `AgentWorkflow` 단일)

장시간 실행되는 AI 작업을 안정적으로 오케스트레이션하려면, 메시지 큐 + workflow 엔진 + managed compute의 조합이 필요합니다. NATS가 메시지를 전달하고, Temporal이 실행을 보장하고, AWS Batch가 실제 연산을 수행합니다. 각자의 역할이 명확하면 전체가 단순해집니다.
