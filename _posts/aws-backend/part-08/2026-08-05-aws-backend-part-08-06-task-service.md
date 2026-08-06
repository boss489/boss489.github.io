---
title: "Chapter 06. Task and Service"
permalink: /aws-backend/part-08/06-task-service/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Task and Service
## ECS의 실행 단위와 유지 단위

> **학습 목표**
>
> - Task Definition, Task, Service의 관계를 설명할 수 있다.
> - Desired Count의 의미를 이해한다.
> - Service가 배포와 장애 복구에 미치는 영향을 설명할 수 있다.

---

# Task Definition

Task Definition은 컨테이너 실행 설정이다.

Image, CPU, Memory, Port, Environment, Log 설정을 담는다.

---

# Task

Task는 Task Definition을 실제로 실행한 단위다.

컨테이너 한 개 이상을 포함할 수 있다.

---

# Service

Service는 원하는 Task 수를 유지한다.

Task가 죽으면 새 Task를 띄우고, 배포 시 새 버전 Task로 교체한다.

---

# 기억해야 할 내용

Task Definition은 설계도, Task는 실행 인스턴스, Service는 유지 관리자다.


