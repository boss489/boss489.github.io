---
title: "Part 1. Cloud Computing"
permalink: /aws-backend/part-01/
date: 2026-08-05T09:27:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 1. Cloud Computing
## AWS Foundation - 클라우드의 탄생과 AWS의 시작

> 목표
>
> AWS 서비스를 배우기 전에,
> Cloud Computing이 왜 등장했는지,
> AWS가 어떤 문제를 해결하기 위해 만들어졌는지를 이해한다.

---

# Part 목표

이 Part를 마치면 다음 질문에 답할 수 있어야 한다.

- 왜 기업들은 Cloud를 사용하는가?
- On-Premise는 어떤 문제가 있었는가?
- Virtual Machine은 어떻게 동작하는가?
- Hypervisor는 무엇인가?
- AWS는 왜 Region과 AZ를 사용하는가?
- Edge Location은 무엇을 위한 것인가?
- Shared Responsibility Model은 왜 중요한가?

---

# 학습 순서

```
On-Premise
      │
      ▼
Virtualization
      │
      ▼
Hypervisor
      │
      ▼
Cloud Computing
      │
      ▼
AWS
      │
      ▼
Global Infrastructure
      │
      ▼
Region
      │
      ▼
Availability Zone
      │
      ▼
Edge Location
```

---

# Chapter 구성

---

## Chapter 01

### 왜 Cloud가 등장했는가

파일

```
01-why-cloud.md
```

### 목표

Cloud Computing이 탄생한 배경을 이해한다.

### 주요 내용

- 서버는 원래 어떻게 운영했는가?
- IDC란?
- 물리 서버의 문제
- 기업이 겪던 어려움
- Cloud가 해결한 문제

---

## Chapter 02

### On-Premise Infrastructure

파일

```
02-on-premise.md
```

### 목표

Cloud 이전의 인프라를 이해한다.

### 주요 내용

- IDC
- Rack
- Power
- Cooling
- Network
- Storage
- 운영 비용
- 증설 과정

---

## Chapter 03

### Virtualization

파일

```
03-virtualization.md
```

### 목표

왜 VM이 등장했는지 이해한다.

### 주요 내용

- Server Consolidation
- Host OS
- Guest OS
- Virtual Machine
- Resource Isolation

---

## Chapter 04

### Hypervisor

파일

```
04-hypervisor.md
```

### 목표

Hypervisor가 VM을 어떻게 관리하는지 이해한다.

### 주요 내용

- Type 1
- Type 2
- VMware
- Xen
- KVM
- Nitro Hypervisor(Preview)

---

## Chapter 05

### Cloud Computing

파일

```
05-cloud-computing.md
```

### 목표

Cloud Computing의 개념을 이해한다.

### 주요 내용

- Cloud 정의
- Elasticity
- Scalability
- High Availability
- Fault Tolerance

---

## Chapter 06

### Cloud Service Models

파일

```
06-cloud-service-model.md
```

### 목표

Cloud 서비스 모델을 이해한다.

### 주요 내용

- IaaS
- PaaS
- SaaS
- FaaS
- Managed Service

---

## Chapter 07

### AWS 소개

파일

```
07-introduction-to-aws.md
```

### 목표

AWS가 어떤 서비스인지 이해한다.

### 주요 내용

- AWS 역사
- AWS 철학
- AWS 서비스 분류
- AWS Console
- AWS CLI

---

## Chapter 08

### AWS Global Infrastructure

파일

```
08-global-infrastructure.md
```

### 목표

AWS가 전 세계에 인프라를 어떻게 배치하는지 이해한다.

### 주요 내용

- Region
- AZ
- Local Zone
- Wavelength
- Global Network

---

## Chapter 09

### Region과 Availability Zone

파일

```
09-region-and-az.md
```

### 목표

Region과 AZ를 이해한다.

### 주요 내용

- Region
- AZ
- 장애 격리
- Multi AZ
- Latency

---

## Chapter 10

### Edge Network

파일

```
10-edge-network.md
```

### 목표

전 세계 사용자에게 빠르게 서비스를 제공하는 방법을 이해한다.

### 주요 내용

- Edge Location
- CloudFront
- Global Accelerator
- DNS

---

## Chapter 11

### Shared Responsibility Model

파일

```
11-shared-responsibility.md
```

### 목표

AWS와 사용자의 책임을 이해한다.

### 주요 내용

- Security of the Cloud
- Security in the Cloud
- IAM Preview
- Patch
- Encryption

---

## Chapter 12

### Well-Architected Framework (Preview)

파일

```
12-well-architected-preview.md
```

### 목표

AWS가 권장하는 아키텍처 철학을 이해한다.

### 주요 내용

- Operational Excellence
- Security
- Reliability
- Performance Efficiency
- Cost Optimization
- Sustainability

---

## Chapter 13

### Part 1 Summary

파일

```
13-summary.md
```

### 목표

Part 1 전체를 정리한다.

### 내용

- 핵심 개념 정리
- 그림으로 복습
- 실무 포인트
- 자주 하는 실수
- 다음 Part 예고

---

## Chapter 14

### Interview Questions

파일

```
14-interview.md
```

### 목표

Part 1 내용을 면접 수준으로 정리한다.

### 내용

- 예상 질문
- 모범 답안
- 심화 질문

---

# 실습

Part 1에서는 AWS 리소스를 생성하지 않는다.

대신 다음을 수행한다.

- AWS 계정 생성
- IAM 사용자 생성
- AWS Console 둘러보기
- AWS CLI 설치
- AWS CLI 인증 설정

---

# Part 1 완료 후

다음 내용을 설명할 수 있어야 한다.

- Cloud가 왜 필요한가?
- AWS는 무엇인가?
- Region과 AZ의 차이
- Hypervisor란?
- IaaS와 PaaS의 차이
- Shared Responsibility Model

---

다음 Part

→ **Part 2. AWS Networking**