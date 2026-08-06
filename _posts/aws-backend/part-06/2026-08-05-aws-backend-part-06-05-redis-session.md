---
title: "Chapter 05. Redis Session"
permalink: /aws-backend/part-06/05-redis-session/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Redis Session
## 여러 서버가 세션을 공유하는 방식

> **학습 목표**
>
> - 서버 메모리 세션의 한계를 설명할 수 있다.
> - Redis Session의 역할을 이해한다.
> - 세션 저장소 장애가 로그인에 미치는 영향을 설명할 수 있다.

---

# 왜 Redis Session을 쓰는가

서버가 여러 대면 사용자의 요청이 매번 같은 서버로 가지 않을 수 있다.

세션을 서버 메모리에 저장하면 다른 서버가 세션을 찾지 못한다.

Redis에 세션을 저장하면 여러 서버가 같은 세션 저장소를 공유할 수 있다.

---

# 장점

- Scale Out에 유리하다.
- 서버 재시작 시 세션 손실을 줄일 수 있다.
- Sticky Session 의존도를 낮출 수 있다.

---

# 주의할 점

Redis 장애는 로그인 상태에 영향을 줄 수 있다.

세션 TTL, Redis 고가용성, 장애 시 사용자 경험을 함께 고려해야 한다.

---

# 기억해야 할 내용

Redis Session은 여러 서버가 세션을 공유하기 위한 방식이다.

JWT와 비교해 서버 측 강제 만료가 쉽지만 중앙 저장소 의존성이 생긴다.


