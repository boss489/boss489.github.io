---
title: "Chapter 07. Failover"
permalink: /aws-backend/part-05/07-failover/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Failover
## 장애 시 다른 DB로 전환하기

> **학습 목표**
>
> - Failover의 의미를 설명할 수 있다.
> - Failover 중 애플리케이션에 발생할 수 있는 영향을 이해한다.
> - 연결 풀과 재시도 설정의 중요성을 설명할 수 있다.

---

# Failover란?

Failover는 Primary DB에 장애가 발생했을 때 Standby 또는 다른 노드로 역할을 넘기는 과정이다.

Multi-AZ 구성에서 주로 사용된다.

---

# 애플리케이션 영향

Failover 중에는 짧은 연결 실패가 발생할 수 있다.

애플리케이션은 DB 연결을 다시 맺을 수 있어야 한다.

---

# 확인할 것

- Connection Pool 설정
- DB Endpoint 사용 여부
- 재시도 정책
- Transaction 처리
- 장애 시간 동안 사용자 영향

---

# 기억해야 할 내용

Failover는 무중단을 보장하지 않는다.

짧은 장애를 견디도록 애플리케이션과 연결 풀을 설정해야 한다.


