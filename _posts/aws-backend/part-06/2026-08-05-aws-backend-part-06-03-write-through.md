---
title: "Chapter 03. Write Through"
permalink: /aws-backend/part-06/03-write-through/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Write Through
## 저장 시 캐시도 함께 갱신하는 패턴

> **학습 목표**
>
> - Write Through의 흐름을 설명할 수 있다.
> - Cache Aside와 차이를 구분할 수 있다.
> - 쓰기 지연과 장애 처리 이슈를 이해한다.

---

# Write Through란?

Write Through는 데이터를 저장할 때 DB와 캐시를 함께 갱신하는 방식이다.

읽기 시 캐시 적중률을 높일 수 있다.

---

# 장점

- 캐시가 최신 상태에 가깝다.
- 읽기 성능이 안정적이다.

---

# 단점

- 쓰기 경로가 복잡해진다.
- DB 저장은 성공했는데 캐시 갱신이 실패하는 상황을 처리해야 한다.
- 쓰기 요청이 느려질 수 있다.

---

# 기억해야 할 내용

Write Through는 읽기 성능을 위해 쓰기 경로의 복잡도를 받아들이는 방식이다.

단순한 서비스는 Cache Aside가 더 낫다.


