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

---

# 캐시 설계 확인 순서

1. 이 데이터는 얼마 동안 오래되어도 되는가?
2. Cache miss가 동시에 발생해도 DB가 견딜 수 있는가?
3. 쓰기 뒤 캐시를 갱신할지 삭제할지 결정했는가?
4. Redis가 느리거나 내려가도 요청을 DB로 우회할 수 있는가?
5. 중복 실행은 Lock이 아니라 DB 제약으로 막을 수 없는가?

캐시는 정답을 더 빨리 주는 장치가 아니라, 허용한 범위 안에서 오래된 값을 빠르게 주는 장치다.

