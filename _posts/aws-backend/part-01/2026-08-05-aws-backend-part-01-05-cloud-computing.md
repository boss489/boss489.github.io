---
title: "Chapter 05. Cloud Computing"
permalink: /aws-backend/part-01/05-cloud-computing/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Cloud Computing
## 필요한 자원을 필요한 만큼 사용하는 방식

> **학습 목표**
>
> - Cloud Computing의 정의를 설명할 수 있다.
> - Elasticity, Scalability, High Availability, Fault Tolerance를 구분할 수 있다.
> - Cloud가 애플리케이션 아키텍처에 주는 영향을 이해한다.

---

# Cloud Computing이란?

Cloud Computing은 컴퓨팅 자원을 네트워크를 통해 필요한 만큼 사용하는 방식이다.

서버를 직접 구매하는 대신 API나 Console로 리소스를 생성하고 삭제한다.

대표 자원은 다음과 같다.

- Compute
- Storage
- Network
- Database
- Security
- Monitoring

---

# Elasticity

Elasticity는 필요할 때 늘리고 필요 없을 때 줄이는 능력이다.

트래픽이 많을 때 서버를 늘리고, 트래픽이 줄면 서버를 줄인다.

비용과 성능을 동시에 맞추기 위한 핵심 개념이다.

---

# Scalability

Scalability는 더 많은 부하를 처리할 수 있도록 확장 가능한 구조를 말한다.

대표 방식은 두 가지다.

| 방식 | 설명 |
|---|---|
| Scale Up | 더 큰 서버로 바꿈 |
| Scale Out | 서버 수를 늘림 |

Cloud에서는 보통 Scale Out을 많이 사용한다.

---

# High Availability

High Availability는 일부 장애가 발생해도 서비스가 계속 동작하는 구조다.

AWS에서는 여러 AZ에 리소스를 배치해 가용성을 높인다.

---

# Fault Tolerance

Fault Tolerance는 장애가 발생해도 시스템이 정상 동작을 유지하는 능력이다.

High Availability보다 더 강한 개념이며, 장애를 전제로 설계한다.

---

# 기억해야 할 내용

- Cloud는 리소스를 API로 사용하는 방식이다.
- Elasticity는 수요에 따라 늘고 줄어드는 능력이다.
- Scalability는 더 큰 부하를 처리할 수 있는 구조다.
- AWS 아키텍처는 장애를 피하는 것이 아니라 견디도록 설계한다.


