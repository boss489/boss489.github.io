---
title: "Chapter 08. Part 6 Summary"
permalink: /aws-backend/part-06/08-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Part 6 Summary
## Cache 핵심 정리

![Cache patterns](/assets/aws-backend/cache-patterns.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| Cache Aside | 읽기 시 캐시 우선 조회 |
| Write Through | 쓰기 시 캐시도 함께 갱신 |
| TTL | 캐시 데이터 수명 |
| Redis Session | 여러 서버의 세션 공유 |
| Distributed Lock | 여러 서버 간 동시 실행 제어 |
| Pub/Sub | 채널 기반 메시지 전달 |
| ElastiCache | AWS 관리형 Redis |

---

# 실무 포인트

- 캐시는 성능을 높이지만 정합성 비용을 만든다.
- TTL은 데이터 성격별로 다르게 정한다.
- Lock보다 DB 제약 조건이 단순하면 DB를 우선한다.
- Redis 장애가 사용자 경험에 미치는 영향을 정해야 한다.


