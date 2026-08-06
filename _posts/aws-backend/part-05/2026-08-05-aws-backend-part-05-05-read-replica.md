---
title: "Chapter 05. Read Replica"
permalink: /aws-backend/part-05/05-read-replica/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Read Replica
## 읽기 부하를 분산하는 복제본

> **학습 목표**
>
> - Read Replica의 목적을 설명할 수 있다.
> - 복제 지연의 의미를 이해한다.
> - 쓰기와 읽기 트래픽 분리 기준을 설명할 수 있다.

---

# Read Replica란?

Read Replica는 Primary 데이터베이스의 데이터를 복제해 읽기 요청을 처리하는 DB 인스턴스다.

주 목적은 읽기 확장이다.

---

# 복제 지연

Read Replica는 Primary와 완전히 동시에 반영되지 않을 수 있다.

이 지연을 Replication Lag라고 한다.

방금 쓴 데이터를 즉시 읽어야 하는 요청은 Primary를 읽어야 한다.

---

# 사용 예시

- 상품 목록 조회
- 리포트 조회
- 관리자 통계
- 검색 보조 데이터 조회

---

# 기억해야 할 내용

Read Replica는 읽기 성능 확장에 사용한다.

정합성이 중요한 요청은 복제 지연을 고려해야 한다.


