---
title: "Billing 마이크로서비스"
description: "DDD 아키텍처 기반 결제·구독·크레딧 시스템"
date: 2026-01-28
tags: ["DDD", "CQRS", "gRPC", "Stripe", "Python"]
weight: 1
---

## 개요

AI 합성 데이터 플랫폼(DTS)의 과금 체계를 담당하는 마이크로서비스입니다. Credit, Subscription, Payment 3개 Bounded Context를 DDD 패턴으로 설계하고 구현했습니다.

**기간**: 2026.01 ~ 현재
**소속**: 주식회사 큐빅 (CUBIG)

## 기술 스택

Python · SQLAlchemy · Alembic · gRPC · Stripe API · PostgreSQL · Pytest

## 주요 작업

### 🏗 DDD 아키텍처 설계

- Credit, Subscription, Payment 3개 Bounded Context 분리
- Domain Events, Aggregate, Value Object 등 전술적 패턴 적용
- Domain → Application → Infrastructure → Interface 4-layer 구조
- CQRS-lite 패턴으로 커맨드/쿼리 분리, 트랜잭션 원자성 보장

### 💳 Stripe 결제 연동

- Stripe Webhook Handler 구현 및 멱등성 처리
- Payment 기록 저장, 결제 상태 추적
- Stripe secrets를 서비스별 Secrets Manager로 분리

### 🔒 크레딧 동시성 제어

- Pessimistic row lock으로 크레딧 동시 예약 문제 해결
- CreditTransaction audit log로 차감/소비 이력 추적
- E2E 비즈니스 시나리오 테스트 구현

### 🔄 Customer Aggregate 도입 및 대규모 리팩터링

- organization_id → customer_id 통일 (99개 파일, 레이어별 커밋 분리)
- Customer Aggregate 도입으로 PG 매핑 소유권을 도메인 내부로 이동
- Application, Domain, Infrastructure, Interface 레이어별 단계적 마이그레이션

### 📡 gRPC 서비스 간 통신

- gRPC 서버/클라이언트 구현 (Billing ↔ Account 연동)
- Account gRPC 서비스 확장 — Organization 운영 18개 RPC 추가
- Invitation 모델, API, gRPC 서비스 구현

### ✔ 테스트 및 문서화

- 각 레이어별 단위 테스트 + E2E 시나리오 테스트
- MkDocs 기반 DDD 개발 가이드, ERD, API 문서 구축
- CLAUDE.md 기반 팀 협업 규칙 정립
