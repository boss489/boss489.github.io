---
title: "Chapter 02. RDS"
permalink: /aws-backend/part-05/02-rds/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. RDS
## 관계형 데이터베이스 관리형 서비스

> **학습 목표**
>
> - RDS의 역할을 설명할 수 있다.
> - 직접 설치한 DB와 RDS의 차이를 이해한다.
> - RDS 운영에서 사용자가 책임질 부분을 설명할 수 있다.

---

# RDS란?

RDS(Relational Database Service)는 관계형 데이터베이스를 관리형으로 제공하는 AWS 서비스다.

지원 엔진은 MySQL, PostgreSQL, MariaDB, Oracle, SQL Server 등이 있다.

---

# RDS가 대신 해주는 것

- DB 인스턴스 생성
- 스토리지 연결
- 백업 기능
- 패치 일부 자동화
- Multi-AZ 구성
- 모니터링 지표 제공

---

# 사용자가 해야 하는 것

- 스키마 설계
- 인덱스 설계
- 계정과 권한 관리
- 백업 보관 정책 결정
- 성능 튜닝
- 보안 그룹 설정

---

# 기억해야 할 내용

RDS는 운영 부담을 줄여주지만 데이터베이스 설계와 접근 제어 책임은 사용자에게 남는다.


