---
title: "AWS Marketplace 등록 및 코드 보안"
description: "Cython, PyArmor 활용 소스코드 보호 처리"
date: 2026-01-20
tags: ["AWS", "Cython", "PyArmor", "보안", "Docker"]
weight: 2
---

## 개요

DTS 플랫폼을 AWS Marketplace에 등록하기 위해 Python 소스코드 보호 방안을 설계하고 구현했습니다. 고객사에 배포되는 컨테이너 이미지에서 소스코드가 노출되지 않도록 Cython 컴파일과 PyArmor 난독화를 조합한 보안 처리를 적용했습니다.

**기간**: 2026.01
**소속**: 주식회사 큐빅 (CUBIG)

## 기술 스택

Python · Cython · PyArmor · Docker

## 주요 작업

- Cython을 활용한 Python → C 확장 모듈 컴파일로 소스코드 바이너리화
- PyArmor를 활용한 추가 난독화 레이어 적용
- Docker 빌드 파이프라인에 보안 처리 단계 통합
- AWS Marketplace 등록 요구사항 대응
