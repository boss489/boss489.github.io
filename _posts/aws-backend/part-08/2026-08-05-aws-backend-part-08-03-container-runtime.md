---
title: "Chapter 03. Container Runtime"
permalink: /aws-backend/part-08/03-container-runtime/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Container Runtime
## 이미지를 실제 프로세스로 실행하기

> **학습 목표**
>
> - Container와 Image의 차이를 설명할 수 있다.
> - 컨테이너 포트와 환경 변수의 의미를 이해한다.
> - 컨테이너 로그 처리 기준을 설명할 수 있다.

---

# Container란?

Container는 Docker Image를 실행한 프로세스다.

같은 Image로 여러 Container를 만들 수 있다.

---

# 포트

Spring Boot가 컨테이너 내부에서 8080 포트로 실행되면, ECS Task Definition과 Target Group도 이 포트를 알아야 한다.

---

# 환경 변수

DB 주소, Profile, 외부 API 주소 같은 값은 Image 안에 고정하지 않고 환경 변수로 주입한다.

Secret은 별도 Secret Manager나 Parameter Store를 사용하는 것이 좋다.

---

# 기억해야 할 내용

Container는 실행 중인 애플리케이션 프로세스다.

포트, 환경 변수, 로그 설정이 배포 성공을 좌우한다.


