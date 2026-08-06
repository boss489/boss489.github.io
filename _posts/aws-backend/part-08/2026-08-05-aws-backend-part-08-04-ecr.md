---
title: "Chapter 04. ECR"
permalink: /aws-backend/part-08/04-ecr/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. ECR
## Docker Image 저장소

> **학습 목표**
>
> - ECR의 역할을 설명할 수 있다.
> - Image Push/Pull 흐름을 이해한다.
> - Image Scan과 Lifecycle 정책을 설명할 수 있다.

---

# ECR이란?

ECR(Elastic Container Registry)은 AWS의 Docker Image 저장소다.

CI에서 Image를 빌드하고 ECR에 Push하면 ECS가 해당 Image를 Pull해서 실행한다.

![ECS deployment](/assets/aws-backend/ecs-deployment.png)

---

# 운영 기능

- Repository
- Image Tag
- Image Scan
- Lifecycle Policy
- IAM 기반 접근 제어

---

# 기억해야 할 내용

ECR은 컨테이너 배포의 이미지 저장소다.

오래된 Image는 Lifecycle 정책으로 정리한다.


