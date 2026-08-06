---
title: "Chapter 01. Database Overview"
permalink: /aws-backend/part-05/01-database-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Database Overview
## AWS에서 데이터베이스를 운영한다는 것

> **학습 목표**
>
> - 애플리케이션 데이터 저장소의 운영 포인트를 설명할 수 있다.
> - RDS, Aurora, Backup, Failover의 관계를 이해한다.
> - 데이터베이스 장애가 서비스에 주는 영향을 설명할 수 있다.

---

# 전체 구조

Spring Boot 서비스는 보통 관계형 데이터베이스에 핵심 데이터를 저장한다.

AWS에서는 직접 DB를 설치하기보다 RDS나 Aurora 같은 관리형 서비스를 많이 사용한다.

![Database operations](/assets/aws-backend/database-operations.png)

---

# 운영 포인트

- 가용성
- 백업과 복구
- 읽기 확장
- 장애 전환
- 성능 지표
- 접근 제어

---

# 기억해야 할 내용

데이터베이스는 상태를 가진 리소스다.

Compute처럼 쉽게 버리고 새로 만들 수 없으므로 백업, 복구, 장애 전환 전략이 중요하다.


