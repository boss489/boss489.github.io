---
title: "Chapter 01. Compute Overview"
permalink: /aws-backend/part-03/01-compute-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Compute Overview
## 애플리케이션이 실행되는 위치

> **학습 목표**
>
> - AWS Compute 서비스의 역할을 설명할 수 있다.
> - EC2, ECS, Lambda의 차이를 큰 흐름으로 이해한다.
> - Spring Boot 애플리케이션이 어떤 실행 환경 위에서 동작하는지 설명할 수 있다.

---

# Compute란?

Compute는 애플리케이션 코드를 실행하는 자원이다.

백엔드 관점에서는 Spring Boot 프로세스가 실제로 떠 있는 실행 환경이라고 볼 수 있다.

대표 서비스는 다음과 같다.

| 서비스 | 특징 |
|---|---|
| EC2 | 가상 서버를 직접 관리 |
| ECS | 컨테이너를 클러스터에서 실행 |
| Lambda | 함수 단위로 실행 |
| App Runner | 컨테이너 기반 관리형 실행 환경 |

Part 3에서는 가장 기본이 되는 EC2를 중심으로 학습한다.

---

# EC2를 먼저 배우는 이유

ECS, EKS, Lambda를 쓰더라도 Compute의 기본 개념은 EC2에서 시작한다.

서버가 어떻게 생성되고, 디스크가 어떻게 붙고, 네트워크 인터페이스가 어떻게 연결되는지 이해해야 이후 서비스도 쉽게 이해할 수 있다.

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# 기억해야 할 내용

- Compute는 애플리케이션을 실행하는 자원이다.
- EC2는 AWS의 기본 가상 서버 서비스다.
- AMI, EBS, ENI는 EC2를 이해하는 핵심 구성 요소다.
- Docker와 Systemd는 애플리케이션 실행 방식과 관련된다.


