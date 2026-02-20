---
title: "AWS Marketplace 등록 및 코드 보안"
description: "Cython, PyArmor 활용 소스코드 보호 처리"
date: 2026-01-20
tags: ["AWS", "Cython", "PyArmor", "보안", "Docker"]
weight: 2
mermaid: true
---

## 개요

DTS 플랫폼을 AWS Marketplace에 등록하기 위해 Python 소스코드 보호 방안을 설계하고 구현했습니다. 고객사에 배포되는 컨테이너 이미지에서 소스코드가 노출되지 않도록 Cython 컴파일과 PyArmor 난독화를 조합한 보안 처리를 적용했습니다.

**기간**: 2026.01
**소속**: 주식회사 큐빅 (CUBIG)

## 기술 스택

Python · Cython · PyArmor · Docker · AWS Marketplace

---

## 아키텍처

### 코드 보안 파이프라인

<div class="mermaid">
graph LR
SRC[Python Source]
subgraph "Stage 1: Cython 컴파일"
CY[Cython Compiler]
SO[".so 바이너리"]
end
subgraph "Stage 2: PyArmor 난독화"
PA[PyArmor]
OBF[난독화된 모듈]
end
subgraph "Stage 3: 패키징"
DCK[Docker Build]
IMG[Container Image]
end
subgraph "배포"
ECR[ECR]
MKT[AWS Marketplace]
end
SRC --> CY
CY --> SO
SO --> PA
PA --> OBF
OBF --> DCK
DCK --> IMG
IMG --> ECR
ECR --> MKT
</div>

---

## 주요 설계 결정

### 🔒 왜 이중 보안(Cython + PyArmor)인가

Cython만 사용하면 `.so` 파일에서 함수 시그니처와 문자열 상수가 노출됩니다. PyArmor만 사용하면 런타임 오버헤드와 호환성 문제가 있습니다. 두 도구를 조합해서 Cython이 바이너리화를 담당하고 PyArmor가 진입점과 설정 파일의 난독화를 담당하는 구조로 각각의 약점을 보완했습니다.

### 🐳 Docker 멀티스테이지 빌드

보안 처리 과정에서 Cython 빌드 도구, PyArmor 라이센스 등 빌드 전용 의존이 필요합니다. Docker 멀티스테이지 빌드를 적용해 빌드 스테이지에서만 이 도구들을 사용하고, 최종 이미지에는 컴파일된 바이너리와 런타임 의존만 포함시켰습니다. 이미지 크기가 줄고, 빌드 도구 정보가 배포 이미지에 남지 않습니다.

### 📦 AWS Marketplace 요구사항 대응

AWS Marketplace는 컨테이너 이미지에 대해 보안 스캔을 수행하고, 특정 구조를 요구합니다. Marketplace 등록 가이드라인에 맞춰 이미지 태깅, 메타데이터 구성, 취약점 스캔 통과를 위한 베이스 이미지 선택을 진행했습니다.
