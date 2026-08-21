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

## Fargate와 EC2 Launch Type의 차이는 무엇인가요?

Fargate는 ECS가 실행할 서버 용량을 관리하므로 운영 부담이 적습니다. EC2 Launch Type은 인스턴스와 컨테이너 밀도를 직접 조절할 수 있지만 서버 운영 책임이 추가됩니다.

## Task가 실행 중인데 Service 배포가 실패할 수 있나요?

가능합니다. Task가 시작되어도 Target Group Health Check를 통과하지 못하면 Service는 healthy 상태로 보지 않습니다. 애플리케이션 포트, Health Check 경로, 기동 시간을 확인합니다.

## 컨테이너에 상태를 두면 왜 문제가 되나요?

Task는 배포·스케일링·장애 복구 중 언제든 교체될 수 있습니다. 파일, 세션, 업로드 같은 상태는 S3·Redis·DB처럼 Task 밖의 저장소에 둬야 합니다.

