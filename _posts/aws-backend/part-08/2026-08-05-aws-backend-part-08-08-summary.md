---
title: "Chapter 08. Part 8 Summary"
permalink: /aws-backend/part-08/08-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Part 8 Summary
## Container Platform 핵심 정리

![ECS deployment](/assets/aws-backend/ecs-deployment.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| Docker Image | 배포 가능한 실행 패키지 |
| Container | 실행 중인 프로세스 |
| ECR | Image 저장소 |
| ECS Cluster | 컨테이너 실행 논리 공간 |
| Task Definition | 컨테이너 실행 설정 |
| Task | 실제 실행 단위 |
| Service | 원하는 Task 수 유지 |

---

# 실무 포인트

- Image Tag는 추적 가능해야 한다.
- 로그는 CloudWatch Logs로 보낸다.
- Health Check가 배포 안정성을 결정한다.
- 처음에는 Fargate가 운영 부담이 적다.


