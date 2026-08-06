---
title: "Chapter 01. Cache Overview"
permalink: /aws-backend/part-06/01-cache-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Cache Overview
## 느린 조회를 빠르게 만드는 중간 저장소

> **학습 목표**
>
> - Cache를 사용하는 이유를 설명할 수 있다.
> - Redis와 ElastiCache의 역할을 이해한다.
> - 캐시가 항상 정답은 아니라는 점을 설명할 수 있다.

---

# Cache란?

Cache는 자주 읽는 데이터를 더 빠른 저장소에 임시로 저장하는 방식이다.

DB 조회를 줄이고 응답 시간을 낮추기 위해 사용한다.

![Cache patterns](/assets/aws-backend/cache-patterns.png)

---

# 장점과 비용

장점은 빠른 응답과 DB 부하 감소다.

비용은 데이터 정합성 관리다.

캐시에 오래된 데이터가 남을 수 있으므로 TTL과 무효화 전략이 필요하다.

---

# 기억해야 할 내용

캐시는 성능 도구다.

정합성이 더 중요한 데이터에는 신중하게 적용해야 한다.


