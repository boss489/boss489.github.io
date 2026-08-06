---
title: "Chapter 02. Docker Image"
permalink: /aws-backend/part-08/02-docker-image/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Docker Image
## 실행 가능한 애플리케이션 패키지

> **학습 목표**
>
> - Docker Image의 역할을 설명할 수 있다.
> - Image Tag가 배포에 미치는 영향을 이해한다.
> - Spring Boot Image를 만들 때 고려할 점을 설명할 수 있다.

---

# Docker Image란?

Docker Image는 애플리케이션과 실행에 필요한 파일을 묶은 읽기 전용 템플릿이다.

Image를 실행하면 Container가 된다.

---

# Tag

Tag는 Image 버전을 식별한다.

`latest`만 사용하면 어떤 코드가 배포됐는지 추적하기 어렵다.

실무에서는 Git SHA, 빌드 번호, 릴리스 버전을 Tag로 사용한다.

---

# Spring Boot Image

Spring Boot Image에는 다음이 포함된다.

- JRE 또는 JDK
- 애플리케이션 JAR
- 실행 명령
- 환경 변수 기본값

---

# 기억해야 할 내용

Image는 배포 단위다.

운영 배포에는 추적 가능한 Tag를 사용해야 한다.


