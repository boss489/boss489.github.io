---
title: "Chapter 05. Target Group"
permalink: /aws-backend/part-04/05-target-group/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Target Group
## ALB가 요청을 보낼 대상 묶음

> **학습 목표**
>
> - Target Group의 역할을 설명할 수 있다.
> - Target Type의 차이를 이해한다.
> - Target Group과 Health Check의 관계를 설명할 수 있다.

---

# Target Group이란?

Target Group은 ALB가 요청을 전달할 대상들의 묶음이다.

대상은 EC2, IP, Lambda 등이 될 수 있다.

ECS에서는 보통 Task IP가 Target으로 등록된다.

---

# Target Type

| Type | 설명 |
|---|---|
| instance | EC2 인스턴스를 대상으로 등록 |
| ip | IP 주소를 대상으로 등록 |
| lambda | Lambda 함수를 대상으로 등록 |

ECS Fargate는 `ip` 타입을 사용한다.

---

# 포트

Target Group은 대상 포트를 가진다.

Spring Boot가 8080으로 실행되면 Target Group도 8080으로 요청을 보내야 한다.

---

# 기억해야 할 내용

- Target Group은 ALB 뒤의 대상 묶음이다.
- Target Group 포트와 애플리케이션 포트가 맞아야 한다.
- Health Check가 정상인 대상만 트래픽을 받는다.


