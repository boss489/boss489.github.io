---
title: "Part 8. Container Platform"
permalink: /aws-backend/part-08/
date: 2026-08-05T09:09:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 8. Container Platform
## ECS 기반 컨테이너 운영

> 목표
>
> Docker 기반 Spring Boot 서비스를 ECS에서 운영하는 방법을 이해한다.

---

# 학습 내용

- Docker
- Image
- Container
- ECR
- ECS
- Task
- Service
- Cluster

---

# 이 Part에서 연결할 배포 흐름

```text
Source → Docker Image build → ECR push → Task Definition revision
       → ECS Service deployment → Target Group Health Check → Traffic
```

컨테이너화는 실행 환경을 고정하는 일이고, ECS는 그 컨테이너를 원하는 수만큼 실행·교체하는 일이다. 둘을 구분하면 배포 실패 지점도 Image, Task, Service, Load Balancer 순서로 좁힐 수 있다.

# 실무 판단 기준

- Image Tag는 `latest`만 쓰지 않고 배포한 소스 버전을 추적할 수 있어야 한다.
- Task는 가능한 한 상태를 갖지 않게 만들고, 설정·비밀값·로그를 컨테이너 밖의 관리형 서비스로 분리한다.
- Service의 Health Check와 배포 설정은 새 Task가 정상일 때만 트래픽을 받도록 구성한다.

---

# 완료 후 설명할 수 있어야 하는 것

- Docker Image와 Container의 차이
- ECR에 이미지를 저장하고 ECS에서 실행하는 흐름
- ECS Task, Service, Cluster의 관계
- ECS 배포와 스케일링의 기본 구조

---

다음 Part

→ **Part 9. Kubernetes**
