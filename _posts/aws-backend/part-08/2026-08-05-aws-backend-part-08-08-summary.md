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

---

# 배포 실패 확인 순서

1. ECR에 의도한 Image Tag가 존재하고 Task Execution Role이 Pull할 수 있는가?
2. Task Definition의 CPU, Memory, 환경 변수, 포트가 애플리케이션과 맞는가?
3. Container가 시작 직후 종료하지 않고 로그를 남기는가?
4. Target Group의 Health Check 경로·포트가 성공하는가?
5. Service가 새 Task를 정상으로 판단한 뒤 트래픽을 전환하는가?

ECS 배포는 컨테이너가 실행되는 것만으로 완료되지 않는다. Load Balancer 관점에서 정상 응답할 때 완료다.

