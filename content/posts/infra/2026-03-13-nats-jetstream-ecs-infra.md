---
title: "NATS JetStream on AWS ECS — Fargate를 포기한 이유"
date: 2026-03-13
categories: ["infra"]
tags: ["NATS", "JetStream", "AWS", "ECS", "EC2", "EBS", "Cloud Map", "서비스 디스커버리", "메시지 브로커", "마이크로서비스"]
series: ["Python 마이크로서비스에 NATS JetStream 도입기"]
series_order: 1
project: "backend"
source_sessions: []
---

Python 마이크로서비스 간 비동기 통신이 필요했습니다. NATS JetStream을 선택했고, AWS ECS 위에 올려야 했습니다.

## 🏁 요구사항

5개의 마이크로서비스가 ECS Fargate 위에서 돌아가고 있었습니다. 서비스 간 통신은 gRPC로 처리하고 있었지만, 몇 가지 흐름은 동기 호출이 맞지 않았습니다.

- 결제 서비스에서 구독 상태가 바뀌면 외부 분석 시스템에 전달해야 합니다
- 이벤트를 놓치면 안 됩니다 — at-least-once 보장이 필요합니다
- 메시지 브로커 자체의 운영 부담은 최소화하고 싶습니다

Kafka는 오버스펙이었습니다. SQS+SNS 조합은 CloudEvents envelope을 직접 구현해야 하고, subject 기반 라우팅이 안 됩니다. RabbitMQ는 이전 프로젝트에서 써봤지만 클러스터 운영이 번거로웠습니다.

NATS JetStream은 단일 바이너리, 내장 persistence, subject 기반 라우팅, 가벼운 리소스 사용량이 장점이었습니다. Go로 작성되어 ARM64 이미지도 바로 쓸 수 있었습니다.

## 🔧 Fargate vs EC2 — 스토리지가 갈랐습니다

ECS에 NATS를 올리는 첫 번째 결정은 Fargate냐 EC2냐였습니다.

| 항목 | Fargate | EC2 |
|------|---------|-----|
| 스토리지 | EFS (NFS) | EBS (블록) |
| JetStream 호환성 | ❌ stale handle 이슈 | ✅ 권장 |
| 인스턴스 관리 | 없음 | ASG 필요 |
| 비용 (t4g.small) | — | ~$12/월 |

JetStream은 WAL(Write-Ahead Log) 기반으로 동작합니다. 블록 스토리지가 필요합니다.

Fargate는 EBS를 마운트할 수 없습니다. EFS만 가능한데, [nats-io/nats-server#2792](https://github.com/nats-io/nats-server/issues/2792)에서 EFS의 stale NFS handle로 JetStream이 비활성화되는 문제가 보고되었습니다. NFS 프로토콜 특성상 파일 핸들이 만료되면 서버가 스토리지를 잃어버립니다. NATS 측에서도 NFS는 권장하지 않습니다.

결론: NATS만 EC2로 돌리고, 나머지 서비스는 Fargate를 유지합니다. 하이브리드 구성입니다.

## 🧱 Dev 환경 구성

### ECS Task Definition

```
nats:2.10-alpine (ARM64)
├── nats        (메인 컨테이너)
├── nats-box    (디버깅 CLI, sleep infinity)
└── nats-nui    (웹 UI, 포트 31311)
```

t4g.small (2 vCPU, 2GB RAM)에 EBS gp3 10GB를 마운트했습니다. NATS 자체는 가볍습니다. 메인 컨테이너의 리소스 할당은 CPU 128, Memory 256MB면 충분합니다.

Sidecar로 `nats-box`와 `nats-nui`를 함께 띄웠습니다. `nats-box`는 `nats sub`, `nats stream ls` 같은 CLI를 컨테이너 안에서 바로 쓸 수 있어서 디버깅이 빠릅니다. `nats-nui`는 Stream과 Consumer 상태를 웹으로 확인할 수 있는 UI입니다. 둘 다 prod에서는 빼도 됩니다.

### JetStream 설정

```ini
jetstream {
  store_dir: /data
  max_mem: 256MB
  max_file: 8GB
}
max_payload: 1MB
ping_interval: "30s"
ping_max: 3
```

`max_mem` 256MB는 인메모리 캐시 한도입니다. JetStream은 핫 데이터를 메모리에 올려두는데, 이 한도를 넘으면 디스크로 내립니다. t4g.small의 2GB 중 NATS에 256MB를 할당한 셈입니다.

`max_file` 8GB는 EBS 10GB 중 JetStream이 사용할 디스크 한도입니다. 나머지 2GB는 OS와 로그용입니다.

`ping_interval`을 기본값 2분에서 30초로 줄였습니다. ECS 태스크가 종료되거나 네트워크가 끊겼을 때 dead connection을 빨리 감지하기 위해서입니다. 30초 × 3회 = 90초면 stale connection이 정리됩니다.

### 보안 그룹

```
nats-task-sg:
  4222  ← ecs_service_sg       (서비스 → NATS)
  4222  ← VPC CIDR             (SSM 포트포워딩)
  8222  ← VPC CIDR             (모니터링 API)
  31311 ← VPC CIDR             (NUI 웹 UI)
  6222  ← Self                 (Raft — Prod만)
```

4222는 클라이언트 포트입니다. ECS 서비스들의 보안 그룹(ecs_service_sg)에서만 접근을 허용합니다. VPC CIDR 허용은 SSM Session Manager로 포트포워딩해서 로컬에서 `nats sub` 같은 명령을 쓰기 위해서입니다.

8222는 NATS의 모니터링 HTTP API입니다. `/healthz`, `/jsz` 같은 엔드포인트를 제공합니다. ECS 헬스체크가 이 포트의 `/healthz`를 찍습니다.

### 서비스 디스커버리

서비스들이 NATS에 접속할 때 IP를 하드코딩할 수 없습니다. ECS 태스크는 재시작할 때마다 IP가 바뀝니다.

AWS Cloud Map을 사용했습니다. NATS 태스크가 등록되면 프라이빗 DNS 레코드가 자동으로 생성됩니다. 서비스들은 환경변수 하나로 접속합니다.

```
NATS_URL=nats://nats.{namespace}.internal:4222
```

Cloud Map의 DNS TTL은 10초로 설정했습니다. NATS 태스크가 재시작되면 새 IP가 10초 안에 반영됩니다. Route 53 프라이빗 호스팅 존을 직접 관리할 필요가 없습니다.

## 📐 Prod HA 설계

Dev는 단일 노드로 충분하지만, Prod는 가용성이 필요합니다. 3노드 Raft 클러스터를 설계했습니다.

```
AZ-a                AZ-b                AZ-c
┌──────────┐       ┌──────────┐       ┌──────────┐
│ nats-0   │◄─────►│ nats-1   │◄─────►│ nats-2   │
│ EBS 10GB │  6222 │ EBS 10GB │  6222 │ EBS 10GB │
└──────────┘ (Raft)└──────────┘ (Raft)└──────────┘
     ▲                   ▲                   ▲
     └───────── 4222 ────┴── Cloud Map MULTIVALUE ─┘
```

### 왜 3노드인가

NATS JetStream의 Raft 합의 알고리즘은 과반수 투표를 요구합니다. 3노드면 quorum이 2입니다. 1대가 죽어도 나머지 2대가 과반이므로 쓰기가 계속됩니다. 2노드는 1대 장애 시 과반을 만족하지 못해서 의미가 없습니다.

### Cross-AZ 배치

각 노드를 서로 다른 AZ에 배치합니다. AZ-a 전체가 장애를 겪어도 AZ-b + AZ-c의 2노드가 quorum을 유지합니다. EBS는 AZ에 바인딩되므로 각 노드의 디스크도 자연스럽게 분산됩니다.

### JetStream R=3

Stream 생성 시 `num_replicas=3`을 지정하면 모든 메시지가 3노드에 복제됩니다. 리더가 죽으면 팔로워 중 하나가 리더로 선출되고, 메시지 유실 없이 서비스가 계속됩니다.

### Cloud Map MULTIVALUE

Dev에서는 A 레코드 하나가 NATS IP를 가리켰지만, Prod에서는 MULTIVALUE 라우팅을 사용합니다. 하나의 DNS 이름에 3개의 IP가 반환됩니다. NATS 클라이언트 라이브러리는 여러 서버 URL을 받아서 자동으로 failover합니다.

```python
# nats-py는 URL 리스트 또는 comma-separated 문자열을 지원합니다
nc = await nats.connect("nats://nats.{namespace}.internal:4222")
# Cloud Map이 3개 IP를 반환 → 클라이언트가 자동 분산
```

### 비용

| 항목 | 수량 | 월 비용 |
|------|------|--------|
| t4g.small | 3대 | ~$37 |
| EBS gp3 10GB | 3개 | ~$2.4 |
| Cloud Map | — | ~$0.6 |
| **합계** | | **~$40** |

RI(Reserved Instance) 1년 적용 시 ~$28/월까지 내려갑니다. MSK(Managed Kafka)의 최소 구성이 $200+/월인 것과 비교하면 부담이 적습니다.

## 🏁 결과

Dev 환경에서 NATS JetStream이 안정적으로 동작하고 있습니다.

| 항목 | 값 |
|------|-----|
| NATS 이미지 | `nats:2.10-alpine` (ARM64) |
| 인스턴스 | t4g.small (ECS on EC2) |
| 스토리지 | EBS gp3 10GB |
| 서비스 디스커버리 | Cloud Map (프라이빗 DNS) |
| Prod 설계 | 3노드 Raft, Cross-AZ, R=3 |
| Prod 예상 비용 | ~$40/월 (RI 시 ~$28) |

Terraform 코드로 Dev/Prod 모두 관리하고 있습니다. Prod 클러스터는 운영 필요 시점에 `terraform apply` 한 번으로 올릴 수 있습니다.

다음 글에서는 이 인프라 위에 올린 이벤트 파이프라인 — CloudEvents envelope, subject 컨벤션, events-schema 패키지 — 을 다룹니다.
