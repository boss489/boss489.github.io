---
title: "Chapter 09. Interview Questions"
permalink: /aws-backend/part-08/09-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Interview Questions
## ECS 면접 질문

---

## Docker Image와 Container의 차이는 무엇인가요?

Image는 실행 템플릿이고 Container는 Image를 실행한 프로세스입니다.

## ECR은 어떤 역할을 하나요?

Docker Image를 저장하고 ECS가 배포 시 Pull할 수 있게 하는 AWS 저장소입니다.

## ECS Task와 Service의 차이는 무엇인가요?

Task는 실행 단위이고, Service는 원하는 Task 수를 유지하고 배포를 관리하는 단위입니다.

## Task Definition에는 무엇이 들어가나요?

Image, CPU, Memory, Port, Environment, Log 설정 등이 들어갑니다.

## ECS 배포 장애 시 무엇을 확인하나요?

Image Pull 권한, Task Definition, 컨테이너 포트, Target Group, Health Check, 애플리케이션 로그를 확인합니다.


