---
title: "Chapter 08. Docker"
permalink: /aws-backend/part-03/08-docker/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Docker
## 애플리케이션 실행 환경을 이미지로 포장하기

> **학습 목표**
>
> - Docker Image와 Container의 차이를 설명할 수 있다.
> - Spring Boot를 컨테이너로 실행하는 이유를 이해한다.
> - EC2에서 Docker를 사용할 때의 장단점을 설명할 수 있다.

---

# Docker가 필요한 이유

서버마다 Java 버전, 환경 변수, 파일 경로가 다르면 배포가 불안정해진다.

Docker는 애플리케이션 실행 환경을 이미지로 묶어 같은 방식으로 실행하게 만든다.

![Compute runtime](/assets/aws-backend/compute-runtime.png)

---

# Image와 Container

Docker Image는 실행 가능한 템플릿이다.

Container는 Image를 실제로 실행한 프로세스다.

```
Docker Image
  -> docker run
  -> Container
```

---

# Spring Boot 실행

예시는 다음과 같다.

```bash
docker run -p 8080:8080 my-api:latest
```

EC2에서 Docker를 쓰면 서버에 직접 Java와 애플리케이션 파일을 복잡하게 설치하지 않아도 된다.

---

# 주의할 점

Docker만 사용한다고 운영이 완성되는 것은 아니다.

다음도 함께 필요하다.

- 로그 수집
- 컨테이너 재시작 정책
- 이미지 버전 관리
- 환경 변수와 Secret 관리
- Health Check

---

# 기억해야 할 내용

- Image는 실행 템플릿이고 Container는 실행 중인 프로세스다.
- Docker는 실행 환경 차이를 줄인다.
- EC2에서 Docker를 쓰면 배포 단위가 단순해진다.
- 운영에는 로그, 재시작, Secret 관리가 함께 필요하다.


