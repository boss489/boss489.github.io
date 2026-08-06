---
title: "Chapter 08. Part 5 Summary"
permalink: /aws-backend/part-05/08-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Part 5 Summary
## Database 핵심 정리

![Database operations](/assets/aws-backend/database-operations.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| RDS | 관계형 DB 관리형 서비스 |
| Aurora | AWS 최적화 관계형 DB 엔진 |
| Multi-AZ | 고가용성 구성 |
| Read Replica | 읽기 확장 |
| Backup | 복구 지점 보관 |
| Restore | 백업으로 복구 |
| Failover | 장애 시 역할 전환 |

---

# 실무 포인트

- 운영 DB는 Multi-AZ를 우선 검토한다.
- 읽기 확장은 Read Replica로 분리한다.
- 백업은 복구 테스트까지 해야 의미가 있다.
- Failover 중 연결 실패를 애플리케이션이 견뎌야 한다.


