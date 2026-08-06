---
title: "Chapter 07. EKS"
permalink: /aws-backend/part-09/07-eks/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. EKS
## AWS 관리형 Kubernetes

> **학습 목표**
>
> - EKS의 역할을 설명할 수 있다.
> - Control Plane과 Worker Node의 차이를 이해한다.
> - EKS를 선택할 때 고려할 운영 비용을 설명할 수 있다.

---

# EKS란?

EKS(Elastic Kubernetes Service)는 AWS의 관리형 Kubernetes 서비스다.

AWS가 Control Plane 운영을 관리한다.

---

# 구성 요소

- EKS Control Plane
- Worker Node 또는 Fargate
- VPC CNI
- IAM
- ALB Controller
- Cluster Autoscaler 또는 Karpenter

---

# 운영 비용

EKS는 유연하지만 학습과 운영 비용이 크다.

클러스터 업그레이드, Add-on 관리, 권한, 네트워크, 배포 전략을 모두 이해해야 한다.

---

# 기억해야 할 내용

EKS는 Kubernetes를 AWS에서 운영하기 위한 서비스다.

단순한 컨테이너 운영이면 ECS가 더 작고 빠른 선택일 수 있다.


