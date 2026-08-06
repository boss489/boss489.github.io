---
title: "Chapter 06. Distributed Lock"
permalink: /aws-backend/part-06/06-distributed-lock/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Distributed Lock
## 여러 서버 사이의 동시 실행 제어

> **학습 목표**
>
> - Distributed Lock이 필요한 상황을 설명할 수 있다.
> - Redis Lock 사용 시 TTL이 왜 중요한지 이해한다.
> - Lock이 정답이 아닌 경우를 구분할 수 있다.

---

# 왜 필요한가

서버가 여러 대면 같은 작업이 동시에 실행될 수 있다.

예를 들어 쿠폰 발급, 재고 차감, 배치 중복 실행 같은 작업에서 문제가 생길 수 있다.

Distributed Lock은 여러 서버 사이에서 하나만 작업하도록 제어한다.

---

# Redis Lock

Redis의 `SET NX EX` 같은 명령으로 Lock을 구현할 수 있다.

Lock에는 반드시 만료 시간이 있어야 한다.

만료 시간이 없으면 Lock을 잡은 서버가 죽었을 때 영원히 풀리지 않을 수 있다.

---

# 주의할 점

Lock은 시스템을 느리게 만들 수 있다.

DB 제약 조건이나 원자적 업데이트로 해결할 수 있으면 그쪽이 더 단순하다.

---

# 기억해야 할 내용

Distributed Lock은 동시성 제어 도구다.

먼저 DB 트랜잭션과 제약 조건으로 해결 가능한지 확인해야 한다.


