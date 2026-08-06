---
title: "Chapter 08. Session and Keep Alive"
permalink: /aws-backend/part-04/08-session-keep-alive/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Session and Keep Alive
## 요청 연결과 사용자 상태 관리

> **학습 목표**
>
> - 서버 세션과 Sticky Session의 문제를 이해한다.
> - Keep Alive가 연결 재사용에 미치는 영향을 설명할 수 있다.
> - Scale Out 환경에서 세션을 어떻게 다뤄야 하는지 이해한다.

---

# Session

서버 세션은 사용자 상태를 서버 메모리에 저장하는 방식이다.

서버가 여러 대가 되면 사용자의 다음 요청이 다른 서버로 갈 수 있다.

이 경우 세션을 찾지 못해 로그인 상태가 풀릴 수 있다.

---

# Sticky Session

Sticky Session은 같은 사용자의 요청을 같은 서버로 보내는 방식이다.

간단하지만 특정 서버에 부하가 몰릴 수 있고 장애 시 세션이 사라질 수 있다.

실무에서는 Redis Session이나 JWT 같은 대안을 검토한다.

---

# Keep Alive

Keep Alive는 TCP 연결을 요청마다 새로 만들지 않고 재사용하는 방식이다.

연결 생성 비용을 줄이지만, 너무 오래 유지하면 리소스를 점유할 수 있다.

---

# 기억해야 할 내용

- Scale Out 환경에서는 서버 메모리 세션에 주의해야 한다.
- Sticky Session은 단순하지만 장애와 부하 분산에 약점이 있다.
- Keep Alive는 연결 재사용으로 성능을 높인다.


