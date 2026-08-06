---
title: "Chapter 01. Kubernetes Overview"
permalink: /aws-backend/part-09/01-kubernetes-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Kubernetes Overview
## 컨테이너 오케스트레이션 플랫폼

> **학습 목표**
>
> - Kubernetes가 해결하는 문제를 설명할 수 있다.
> - Pod, Deployment, Service, Ingress의 큰 관계를 이해한다.
> - EKS가 Kubernetes 운영에서 맡는 범위를 설명할 수 있다.

---

# Kubernetes란?

Kubernetes는 컨테이너를 여러 서버 위에서 배치, 실행, 복구, 확장하는 오케스트레이션 플랫폼이다.

ECS보다 유연하지만 운영 복잡도도 높다.

![Kubernetes workload](/assets/aws-backend/kubernetes-workload.png)

---

# 핵심 객체

- Pod
- ReplicaSet
- Deployment
- Service
- Ingress

---

# 기억해야 할 내용

Kubernetes는 컨테이너 운영 표준에 가깝다.

단순한 AWS 컨테이너 운영은 ECS가 더 가벼울 수 있다.


