---
title: "Chapter 06. ALB Controller"
permalink: /aws-backend/part-09/06-alb-controller/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. ALB Controller
## Kubernetes와 AWS ALB 연결하기

> **학습 목표**
>
> - ALB Controller의 역할을 설명할 수 있다.
> - Ingress가 AWS ALB로 변환되는 흐름을 이해한다.
> - IAM 권한과 Annotation이 필요한 이유를 설명할 수 있다.

---

# ALB Controller란?

AWS Load Balancer Controller는 Kubernetes Ingress나 Service 설정을 보고 AWS ALB/NLB를 생성하고 관리한다.

Kubernetes 객체와 AWS 리소스를 연결하는 다리 역할을 한다.

---

# 동작 흐름

1. 사용자가 Ingress를 만든다.
2. Controller가 Ingress를 감지한다.
3. AWS API로 ALB, Listener, Target Group을 만든다.
4. Pod 또는 Node를 Target으로 등록한다.

---

# 기억해야 할 내용

ALB Controller는 Kubernetes 선언을 AWS 로드밸런서 리소스로 바꾼다.

IAM 권한과 Annotation 설정이 중요하다.


