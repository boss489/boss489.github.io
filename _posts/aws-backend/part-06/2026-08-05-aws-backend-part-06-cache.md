---
title: "Part 6. Cache"
permalink: /aws-backend/part-06/
date: 2026-08-05T09:11:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 6. Cache
## Redis 기반 성능 개선

> 목표
>
> Redis와 ElastiCache를 활용해 Spring Boot 서비스의 성능을 개선하는 방법을 이해한다.

---

# 학습 내용

- Cache Aside
- Write Through
- TTL
- Session
- Distributed Lock
- Pub/Sub
- ElastiCache

---

# 이 Part에서 다룰 데이터 흐름

```text
Read:  Application → Redis → Cache miss면 DB → Redis 저장
Write: Application → DB → Cache 갱신 또는 삭제
```

캐시는 DB를 대체하는 저장소가 아니라 읽기 부하와 응답 시간을 줄이기 위한 계층이다. 따라서 적중률보다 먼저 데이터가 오래되어도 되는지와 Redis 장애 시의 동작을 정해야 한다.

# 실무 판단 기준

- Cache Aside는 가장 단순한 기본값이지만, 캐시 삭제·갱신 시점을 설계해야 한다.
- TTL은 메모리 회수 장치이면서 오래된 데이터를 허용하는 기간이다. 모든 키에 같은 값을 쓰지 않는다.
- 중복 실행 방지가 필요할 때도 DB 제약 조건이나 트랜잭션으로 해결되는지 먼저 확인한다.

---

# 완료 후 설명할 수 있어야 하는 것

- Cache Aside와 Write Through의 차이
- TTL을 정해야 하는 이유
- Redis Session과 Distributed Lock의 사용 기준
- ElastiCache 운영 시 고려할 장애 지점

---

다음 Part

→ **Part 7. Object Storage**
