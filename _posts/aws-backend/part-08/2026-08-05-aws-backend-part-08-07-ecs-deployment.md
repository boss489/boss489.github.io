---
title: "Chapter 07. ECS Deployment"
permalink: /aws-backend/part-08/07-ecs-deployment/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. ECS Deployment
## 새 컨테이너 이미지를 서비스에 반영하기

> **학습 목표**
>
> - ECS 배포 흐름을 설명할 수 있다.
> - ALB Health Check가 배포에 미치는 영향을 이해한다.
> - Rolling Deployment의 기본 동작을 설명할 수 있다.

---

# 배포 흐름

1. Docker Image를 빌드한다.
2. ECR에 Push한다.
3. 새 Task Definition Revision을 만든다.
4. ECS Service를 업데이트한다.
5. 새 Task가 Health Check를 통과하면 트래픽을 받는다.

![ECS deployment](/assets/aws-backend/ecs-deployment.png)

---

# Rolling Deployment

Rolling Deployment는 새 Task를 띄우고 정상 확인 후 기존 Task를 줄이는 방식이다.

Health Check가 실패하면 배포가 멈추거나 롤백될 수 있다.

---

# 기억해야 할 내용

ECS 배포 성공은 Image, Task Definition, Health Check, Target Group 설정이 모두 맞아야 한다.


