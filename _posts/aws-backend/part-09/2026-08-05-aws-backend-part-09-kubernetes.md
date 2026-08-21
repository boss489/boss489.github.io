---
title: "Part 9. Kubernetes"
permalink: /aws-backend/part-09/
date: 2026-08-05T09:08:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 9. Kubernetes
## EKS 기반 컨테이너 오케스트레이션

> 목표
>
> Kubernetes와 EKS의 기본 구조를 이해한다.

---

# 학습 내용

- Kubernetes
- Pod
- ReplicaSet
- Deployment
- Service
- Ingress
- ALB Controller
- EKS

---

# 이 Part에서 연결할 배포 흐름

```text
Deployment → ReplicaSet → Pod
Client → Ingress → ALB Controller → Service → Pod
```

Kubernetes는 컨테이너 하나를 실행하는 도구가 아니라 원하는 상태를 선언하고, 실제 상태가 그 선언과 같도록 계속 맞추는 플랫폼이다. Pod, Service, Ingress를 각각 배포·내부 접근·외부 HTTP 진입점으로 나누어 이해한다.

# 실무 판단 기준

- 직접 Pod를 생성하지 않고 Deployment로 배포·롤백·복제 수를 관리한다.
- Service가 Pod IP 변경을 숨기지만, 실제 트래픽을 받으려면 readiness 설정도 맞아야 한다.
- EKS는 Kubernetes 생태계가 필요한 경우에 선택하고, 단순한 컨테이너 운영에는 ECS가 더 적은 운영 부담일 수 있다.

---

# 완료 후 설명할 수 있어야 하는 것

- Pod, ReplicaSet, Deployment의 관계
- Kubernetes Service와 Ingress의 역할
- ALB Controller가 AWS 리소스를 연결하는 방식
- ECS와 EKS를 선택하는 기준

---

다음 Part

→ **Part 10. Serverless**
