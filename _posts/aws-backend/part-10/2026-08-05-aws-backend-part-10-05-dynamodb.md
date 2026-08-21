---
title: "Chapter 05. DynamoDB"
permalink: /aws-backend/part-10/05-dynamodb/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. DynamoDB
## 서버리스 NoSQL 데이터베이스

> **학습 목표**
>
> - DynamoDB의 기본 특징을 설명할 수 있다.
> - Partition Key와 Sort Key의 역할을 이해한다.
> - RDB와 다른 설계 방식을 설명할 수 있다.

---

# DynamoDB란?

DynamoDB는 AWS의 관리형 NoSQL 데이터베이스다.

서버리스 아키텍처에서 Lambda와 함께 자주 사용한다.

---

# 상세 학습 목차

- [Part 1. 핵심 개념과 Key]({{ '/aws-backend/part-10/05-dynamodb/01-concepts-and-keys/' | relative_url }})
- [Part 2. Access Pattern과 Single Table Design]({{ '/aws-backend/part-10/05-dynamodb/02-access-patterns/' | relative_url }})
- [Part 3. DynamoDBMapper, Repository, 쓰기 동시성]({{ '/aws-backend/part-10/05-dynamodb/03-mapper-and-concurrency/' | relative_url }})
- [Part 4. 성능, 비용, Capacity, 운영]({{ '/aws-backend/part-10/05-dynamodb/04-capacity-and-operations/' | relative_url }})
- [Part 5. 고급 데이터 모델링 패턴]({{ '/aws-backend/part-10/05-dynamodb/05-advanced-modeling/' | relative_url }})
- [Part 6. AWS SDK v2 Enhanced Client]({{ '/aws-backend/part-10/05-dynamodb/06-enhanced-client/' | relative_url }})
- [Part 7. 종합 설계, 코드 리뷰, 실전 문제]({{ '/aws-backend/part-10/05-dynamodb/07-practice/' | relative_url }})

---

# Key 설계

DynamoDB는 Key 설계가 중요하다.

- Partition Key는 데이터를 나누는 기준이다.
- Sort Key는 같은 Partition 안에서 정렬과 범위 조회에 사용한다.

조회 패턴을 먼저 정하고 테이블을 설계해야 한다.

---

# RDB와 차이

RDB처럼 자유롭게 Join하는 방식이 아니다.

필요한 조회 패턴에 맞게 데이터를 중복 저장하는 경우가 있다.

---

# 기억해야 할 내용

DynamoDB는 조회 패턴 중심으로 설계한다.

RDB 모델을 그대로 옮기면 성능과 비용 문제가 생길 수 있다.

