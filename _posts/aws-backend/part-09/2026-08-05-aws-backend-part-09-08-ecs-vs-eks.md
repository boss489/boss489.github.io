---
title: "Chapter 08. ECS vs EKS"
permalink: /aws-backend/part-09/08-ecs-vs-eks/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. ECS vs EKS
## 어떤 컨테이너 플랫폼을 선택할 것인가

> **학습 목표**
>
> - ECS와 EKS의 선택 기준을 설명할 수 있다.
> - 운영 복잡도와 유연성의 trade-off를 이해한다.
> - 팀 상황에 맞는 선택 기준을 세울 수 있다.

---

# ECS가 맞는 경우

- AWS 중심으로 운영한다.
- 컨테이너 운영을 단순하게 시작하고 싶다.
- Kubernetes 생태계가 꼭 필요하지 않다.
- 운영 인력이 많지 않다.

---

# EKS가 맞는 경우

- Kubernetes 표준이 필요하다.
- 여러 클라우드 또는 온프레미스와 일관성을 원한다.
- 복잡한 오케스트레이션 요구가 있다.
- 팀이 Kubernetes 운영 경험을 갖고 있다.

---

# 기억해야 할 내용

EKS가 더 강력하다고 항상 더 좋은 선택은 아니다.

필요한 운영 능력과 팀의 숙련도를 같이 봐야 한다.


