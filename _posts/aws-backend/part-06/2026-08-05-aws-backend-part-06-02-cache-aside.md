---
title: "Chapter 02. Cache Aside"
permalink: /aws-backend/part-06/02-cache-aside/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Cache Aside
## 애플리케이션이 캐시를 직접 채우는 패턴

> **학습 목표**
>
> - Cache Aside 흐름을 설명할 수 있다.
> - Cache Miss 상황에서 DB를 조회하는 과정을 이해한다.
> - 캐시 무효화가 필요한 이유를 설명할 수 있다.

---

# 흐름

Cache Aside는 애플리케이션이 캐시를 먼저 조회하고, 없으면 DB를 조회한 뒤 캐시에 저장하는 방식이다.

```
read cache
  -> miss
  -> read database
  -> save cache
```

---

# 장점

- 구현이 단순하다.
- 필요한 데이터만 캐시에 올라간다.
- 장애 시 DB로 fallback할 수 있다.

---

# 주의할 점

데이터 변경 시 캐시를 삭제하거나 갱신해야 한다.

삭제를 놓치면 사용자는 오래된 데이터를 볼 수 있다.

---

# 기억해야 할 내용

Cache Aside는 가장 흔한 읽기 캐시 패턴이다.

쓰기 경로에서 캐시 무효화를 함께 설계해야 한다.


