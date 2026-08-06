---
title: "Chapter 04. Multi AZ"
permalink: /aws-backend/part-05/04-multi-az/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Multi AZ
## 데이터베이스 고가용성 구성

> **학습 목표**
>
> - Multi-AZ의 목적을 설명할 수 있다.
> - Standby와 Failover의 관계를 이해한다.
> - Multi-AZ가 읽기 확장 목적이 아님을 설명할 수 있다.

---

# Multi-AZ란?

Multi-AZ는 데이터베이스를 여러 Availability Zone에 배치해 장애에 대비하는 구성이다.

Primary에 장애가 발생하면 Standby로 Failover한다.

![Database operations](/assets/aws-backend/database-operations.png)

---

# 목적

Multi-AZ의 주 목적은 고가용성이다.

읽기 성능을 늘리는 목적은 Read Replica가 담당한다.

---

# 장애 상황

Failover는 다음 상황에서 발생할 수 있다.

- DB 인스턴스 장애
- AZ 장애
- OS 패치
- 네트워크 장애

---

# 기억해야 할 내용

Multi-AZ는 읽기 확장이 아니라 장애 대비 구성이다.

운영 DB는 가능하면 Multi-AZ를 기본으로 검토한다.


