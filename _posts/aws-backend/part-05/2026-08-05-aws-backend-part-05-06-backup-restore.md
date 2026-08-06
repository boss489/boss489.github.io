---
title: "Chapter 06. Backup and Restore"
permalink: /aws-backend/part-05/06-backup-restore/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Backup and Restore
## 장애 이후 데이터를 되돌리는 기준

> **학습 목표**
>
> - Backup과 Restore의 차이를 설명할 수 있다.
> - RPO와 RTO의 의미를 이해한다.
> - 운영 DB 백업 정책의 기본 요소를 설명할 수 있다.

---

# Backup

Backup은 장애나 실수에 대비해 데이터를 복구 가능한 형태로 보관하는 것이다.

RDS는 자동 백업과 스냅샷을 제공한다.

---

# Restore

Restore는 백업을 이용해 데이터베이스를 복구하는 작업이다.

복구는 보통 기존 DB를 되돌리는 것이 아니라 새로운 DB 인스턴스를 만드는 방식으로 진행된다.

---

# RPO와 RTO

| 개념 | 의미 |
|---|---|
| RPO | 얼마나 최근 시점까지 복구해야 하는가 |
| RTO | 얼마나 빨리 복구해야 하는가 |

---

# 기억해야 할 내용

백업은 설정만으로 끝나지 않는다.

복구 리허설을 해봐야 실제 장애 때 사용할 수 있다.


