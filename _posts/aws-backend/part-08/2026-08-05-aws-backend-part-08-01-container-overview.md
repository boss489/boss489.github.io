---
title: "Chapter 01. Container Overview"
permalink: /aws-backend/part-08/01-container-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Container Overview
## 애플리케이션 실행 환경을 표준화하기

> **학습 목표**
>
> - 컨테이너를 사용하는 이유를 설명할 수 있다.
> - Docker, ECR, ECS의 관계를 이해한다.
> - EC2 직접 운영과 ECS 운영의 차이를 설명할 수 있다.

---

# 전체 흐름

Spring Boot를 Docker Image로 만들고 ECR에 저장한 뒤 ECS에서 Task로 실행한다.

![ECS deployment](/assets/aws-backend/ecs-deployment.png)

---

# 왜 컨테이너를 쓰는가

- 실행 환경 차이를 줄인다.
- 배포 단위를 이미지로 고정한다.
- 서버에 직접 파일을 복사하는 배포를 줄인다.
- ECS 같은 플랫폼에서 스케일링하기 쉽다.

---

# 기억해야 할 내용

컨테이너는 애플리케이션 실행 단위이고, ECS는 컨테이너 운영 플랫폼이다.


