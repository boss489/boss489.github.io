---
title: "Part 5. Database"
permalink: /aws-backend/part-05/
date: 2026-08-05T09:12:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 5. Database
## AWS 데이터베이스 운영

> 목표
>
> AWS에서 관계형 데이터베이스를 안정적으로 운영하는 방법을 이해한다.

---

# 학습 내용

- RDS
- Aurora
- Multi AZ
- Read Replica
- Backup
- Restore
- Failover

---

# 이 Part에서 구분할 운영 목적

```text
쓰기 가용성 → Multi-AZ
읽기 확장   → Read Replica
데이터 복구 → Backup / Restore
엔진 선택   → RDS 또는 Aurora
```

모든 구성이 성능을 높이는 것은 아니다. Multi-AZ는 장애 대비이고 Read Replica는 읽기 분산이다. 요구사항을 가용성, 성능, 복구 중 어디에 두는지 먼저 정해야 한다.

# 실무 판단 기준

- 운영 데이터베이스는 장애 시 허용 가능한 중단 시간(RTO)과 손실 범위(RPO)를 먼저 합의한다.
- Failover와 복구는 콘솔 설정만으로 끝나지 않는다. 애플리케이션의 연결 재시도와 Connection Pool 동작까지 확인한다.
- 백업 정책은 실제 Restore 연습으로 검증한다.

---

# 완료 후 설명할 수 있어야 하는 것

- RDS와 Aurora의 차이
- Multi AZ와 Read Replica의 목적
- Backup과 Restore의 운영 기준
- Failover가 발생하는 상황과 영향

---

다음 Part

→ **Part 6. Cache**
