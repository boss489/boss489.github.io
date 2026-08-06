---
title: "Chapter 03. ReplicaSet and Deployment"
permalink: /aws-backend/part-09/03-replicaset-deployment/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. ReplicaSet and Deployment
## 원하는 Pod 수를 유지하고 배포하기

> **학습 목표**
>
> - ReplicaSet과 Deployment의 역할을 설명할 수 있다.
> - Deployment가 배포 단위로 사용되는 이유를 이해한다.
> - Rolling Update 흐름을 설명할 수 있다.

---

# ReplicaSet

ReplicaSet은 원하는 Pod 개수를 유지한다.

Pod가 죽으면 새 Pod를 만든다.

---

# Deployment

Deployment는 ReplicaSet을 관리하고 배포 전략을 제공한다.

일반적으로 사용자는 Pod나 ReplicaSet을 직접 만들기보다 Deployment를 만든다.

---

# Rolling Update

Deployment는 새 버전 Pod를 조금씩 늘리고 기존 버전 Pod를 줄일 수 있다.

Readiness Probe가 실패하면 트래픽을 받지 않게 할 수 있다.

---

# 기억해야 할 내용

ReplicaSet은 개수 유지, Deployment는 배포 관리다.

실무 배포 단위는 Deployment다.


