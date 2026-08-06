---
title: "Chapter 05. ECS Cluster"
permalink: /aws-backend/part-08/05-ecs-cluster/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. ECS Cluster
## 컨테이너를 실행할 논리적 묶음

> **학습 목표**
>
> - ECS Cluster의 의미를 설명할 수 있다.
> - Fargate와 EC2 Launch Type의 차이를 이해한다.
> - Cluster가 직접 애플리케이션을 실행하는 것은 아님을 설명할 수 있다.

---

# ECS Cluster란?

ECS Cluster는 컨테이너 실행 환경을 묶는 논리적 단위다.

Cluster 안에서 Service와 Task가 실행된다.

---

# Fargate와 EC2

| 방식 | 설명 |
|---|---|
| Fargate | 서버 관리를 AWS에 맡김 |
| EC2 | 사용자가 EC2 Capacity를 관리 |

처음에는 Fargate가 단순하다.

---

# 기억해야 할 내용

Cluster는 실행 공간이고, 실제 애플리케이션은 Task로 실행된다.


