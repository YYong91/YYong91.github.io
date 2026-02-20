---
title: "DTS 플랫폼 인프라 및 DevOps"
description: "Terraform IaC, ECS 배포, CI/CD, 문서 자동화"
date: 2026-01-06
tags: ["Terraform", "AWS", "ECS", "K3s", "CI/CD", "MkDocs"]
weight: 3
---

## 개요

DTS 플랫폼의 인프라를 Terraform으로 코드화하고, 서비스 배포 파이프라인 및 내부 문서 시스템을 구축했습니다.

**기간**: 2026.01 ~ 현재
**소속**: 주식회사 큐빅 (CUBIG)

## 기술 스택

Terraform · AWS (ECS, Fargate, Lambda, ALB, Cognito, Secrets Manager) · Docker · K3s · GitHub Actions · CodePipeline · MkDocs · OPA

## 주요 작업

### 🚀 인프라 구축

- Billing 서비스 ECS Task/Service 정의 및 배포
- API Gateway 등록, OPA 기반 인가 정책(PEP/PDP) 설정
- Cognito 인증 연동 (세션 기반 인증 인프라 구성)
- Secrets Manager를 활용한 서비스별 시크릿 분리
- Infisical(시크릿 관리 도구) 도입 및 팀 가이드 작성

### 🔄 CI/CD

- CodeBuild + CodePipeline 기반 빌드/배포 파이프라인
- GitHub Actions → K3s 배포 파이프라인 (MkDocs 문서 사이트)
- 문서 CI를 Windows Docker → Linux K3s로 마이그레이션

### 📚 문서 자동화

- MkDocs 기반 내부 문서 사이트 구축 (ERD, API, DDD 가이드)
- CloudFront를 통한 API docs 배포
- /docs-sync 스킬을 활용한 네비게이션 자동 동기화
