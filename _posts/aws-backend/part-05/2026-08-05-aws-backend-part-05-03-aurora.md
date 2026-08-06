---
title: "Chapter 03. Aurora"
permalink: /aws-backend/part-05/03-aurora/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Aurora
## AWS가 만든 관계형 데이터베이스 엔진

> **학습 목표**
>
> - Aurora와 일반 RDS의 차이를 설명할 수 있다.
> - Aurora의 스토리지 구조가 가용성에 주는 영향을 이해한다.
> - Aurora를 선택할 때 고려할 기준을 설명할 수 있다.

---

# Aurora란?

Aurora는 AWS가 제공하는 MySQL/PostgreSQL 호환 관계형 데이터베이스 엔진이다.

일반 RDS보다 AWS 인프라에 더 최적화되어 있다.

---

# 특징

- 스토리지 자동 확장
- 여러 AZ에 복제되는 스토리지 구조
- 빠른 Failover
- Reader Endpoint 제공
- MySQL/PostgreSQL 호환

---

# 선택 기준

Aurora는 성능과 가용성 면에서 장점이 있지만 비용이 더 높을 수 있다.

작은 서비스는 일반 RDS로 시작하고, 트래픽과 운영 요구가 커질 때 Aurora를 검토해도 된다.

---

# 기억해야 할 내용

Aurora는 AWS 최적화 관계형 DB다.

성능, 가용성, 비용을 함께 보고 선택해야 한다.


