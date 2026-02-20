---
title: "멀티채널 상품 관리 플랫폼"
description: "DDD, MSA, RabbitMQ 기반 대규모 비동기 처리 시스템"
date: 2020-07-01
tags: ["DDD", "MSA", "RabbitMQ", "Python", "AWS", "Terraform", "Multi-Tenancy"]
weight: 10
---

## 개요

커머스 데이터를 여러 마켓플레이스에 제공하는 B2B 시스템을 개발하고 운영했습니다. 대규모 데이터를 안정적으로 처리하는 구조를 만드는 데 집중했습니다.

**기간**: 2020.07 ~ 2025.01
**소속**: 주식회사 저스트큐 (JustQ)

## 기술 스택

Python · SQLAlchemy · Alembic · RabbitMQ (AMQP) · ActiveMQ · MySQL · AWS (ECS, Fargate, Lambda) · Docker · Terraform · Jenkins · GitHub Actions · CloudWatch · Datadog

## 주요 작업

### 🏗 Python 기반 DDD 적용 및 도메인 구조 정립

- 상품·주문·재고 도메인을 나누고, 팀 간 용어를 통일해 유지보수성과 협업 효율을 높임
- 도메인 모델과 이벤트로 비즈니스 규칙을 구조화해 기능 변경 시 영향 범위를 줄임

### ⚡ MSA·서버리스 기반 비동기 처리 아키텍처 구축

- ECS, Lambda, Docker 기반 MSA 환경을 구성하고 운영
- 수백만 건의 상품 데이터를 병렬로 처리해 데이터 일관성을 유지
- MQ 이벤트량 기반 Auto Scaling 패턴을 적용해 운영 비용 약 35% 절감

### 📨 RabbitMQ 기반 대규모 비동기 메시징 시스템 운영

- 시간당 30만 건, 하루 600만 건 이상의 메시지를 처리하는 큐 환경 운영
- DLQ, 재시도 전략, 멱등성 보장, Routing Key 전략 등 도입
- ActiveMQ → RabbitMQ 전환 작업을 주도해 장애율을 낮추고 처리량 개선

### 🏢 Multi-Tenancy 구조 설계

- 고객사별 데이터 격리를 구현하고 공통 도메인/전용 리소스를 분리
- 유지보수 비용 약 25% 절감, 신규 고객 온보딩 속도 향상

### 🔌 외부 시스템(OpenAPI) 연동 서버 구축

- 사방넷, 플레이오토 등 외부 시스템과 연동하는 서버 개발
- OpenAPI 스펙 정의와 Swagger 문서화로 협업 구조 개선
- 연동 장애 분석 및 복구 시나리오를 마련해 오류율 약 30% 감소

### 🔧 DevOps 및 IaC 환경 구축

- Terraform을 도입해 인프라 배포를 자동화하고 배포 시간 50% 단축
- Jenkins와 GitHub Actions 기반 CI/CD 파이프라인 구축
- CloudWatch·Datadog 기반 모니터링 환경 개선
