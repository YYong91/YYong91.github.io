---
title: "DTS 플랫폼 인프라 및 DevOps"
description: "Terraform IaC, ECS 배포, CI/CD, 문서 자동화"
date: 2026-01-06
tags: ["Terraform", "AWS", "ECS", "K3s", "CI/CD", "MkDocs"]
weight: 3
mermaid: true
---

## 개요

DTS 플랫폼의 인프라를 Terraform으로 코드화하고, 서비스 배포 파이프라인 및 내부 문서 시스템을 구축했습니다.

**기간**: 2026.01 ~ 현재
**소속**: 주식회사 큐빅 (CUBIG)

## 기술 스택

Terraform · AWS (ECS, Fargate, Lambda, ALB, Cognito, Secrets Manager) · Docker · K3s · GitHub Actions · CodePipeline · MkDocs · OPA

---

## 아키텍처

### 인프라 구성도

<div class="mermaid">
graph TB
subgraph "Client"
WEB[Web App]
end
subgraph "AWS Cloud"
subgraph "Network"
ALB[Application Load Balancer]
end
subgraph "Auth"
COG[Cognito]
OPA[OPA Policy Engine]
end
subgraph "Compute"
ECS1[Billing Service]
ECS2[Core API Service]
ECS3[Worker Service]
end
subgraph "Data"
RDS[(PostgreSQL)]
SM[Secrets Manager]
end
subgraph "Observability"
CW[CloudWatch]
DD[Datadog]
end
end
WEB --> ALB
ALB --> COG
COG -->|"JWT 검증"| OPA
OPA -->|"인가 통과"| ECS1
OPA -->|"인가 통과"| ECS2
ECS1 --> RDS
ECS2 --> RDS
ECS3 --> RDS
SM -->|"시크릿 주입"| ECS1
SM -->|"시크릿 주입"| ECS2
ECS1 --> CW
ECS2 --> CW
CW --> DD
</div>

### CI/CD 파이프라인

<div class="mermaid">
graph LR
subgraph "Source"
GH[GitHub Push]
end
subgraph "Build"
CB[CodeBuild]
DCK[Docker Image Build]
ECR[ECR Registry]
end
subgraph "Deploy"
CP[CodePipeline]
ECS[ECS Rolling Update]
end
subgraph "Docs Pipeline"
GHA[GitHub Actions]
MK[MkDocs Build]
K3S[K3s Deploy]
end
GH --> CB
CB --> DCK
DCK --> ECR
ECR --> CP
CP --> ECS
GH --> GHA
GHA --> MK
MK --> K3S
</div>

---

## 주요 설계 결정

### 🏗 왜 Terraform으로 IaC를 도입했는가

수동 콘솔 작업으로 인프라를 관리하면 환경 간 차이가 누적됩니다. Terraform으로 모든 리소스를 코드화하면서 dev/staging/prod 환경의 일관성을 확보하고, 변경 이력을 Git으로 추적할 수 있게 되었습니다.

### 🔐 OPA 기반 인가 구조

Cognito가 인증을 담당하고, OPA(Open Policy Agent)가 인가를 담당하는 PEP/PDP 분리 구조를 적용했습니다. 정책을 코드(Rego)로 정의해서 인가 규칙 변경 시 서비스 코드를 수정하지 않고 정책 파일만 업데이트하면 됩니다.

### 🔑 시크릿 관리 전략

초기에는 환경 변수로 시크릿을 관리했습니다. 서비스가 늘어나면서 Secrets Manager로 이관하고, 서비스별로 시크릿을 분리했습니다. 이후 Infisical을 도입해 팀원들이 시크릿을 안전하게 조회/수정할 수 있는 UI를 확보했습니다.

### 📚 문서 자동화

MkDocs 기반 내부 문서 사이트를 구축하고, ERD, API 문서, DDD 설계 가이드를 한 곳에서 관리합니다. 초기에는 Windows Docker 환경에서 빌드했으나 불안정해서 Linux K3s로 마이그레이션했습니다. `/docs-sync` 스킬로 네비게이션을 자동 동기화합니다.
