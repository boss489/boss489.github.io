---
title: "Chapter 05. RDS Failure"
permalink: /aws-backend/part-15/05-rds-failure/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. RDS Failure
## 연결, 잠금, 쿼리와 가용성 진단

> **학습 목표**
>
> - 연결 고갈과 느린 쿼리 및 잠금 문제를 구분할 수 있다.
> - failover, storage와 replica lag 징후를 확인할 수 있다.
> - HikariCP와 RDS 증거를 연결해 안전하게 대응할 수 있다.

---

# 실제 장애 징후

API timeout, connection acquisition timeout, transaction 지연과 DB 연결 오류가 대표 징후다.

DBConnections가 상한에 접근하거나 FreeableMemory와 FreeStorageSpace가 감소할 수 있다.

Read Replica 지연이 증가하면 오래된 조회 결과와 읽기 endpoint timeout이 나타날 수 있다.

failover 중에는 기존 TCP 연결이 끊기고 DNS endpoint가 새 Primary를 가리키는 전환 구간이 생긴다.

---

# 정의와 가능한 원인

RDS 장애는 DB 인스턴스가 중단된 경우뿐 아니라 애플리케이션이 필요한 시간 안에 정확한 결과를 얻지 못하는 상태를 포함한다.

connection exhaustion은 애플리케이션 인스턴스 수 증가, pool 과대 설정, connection leak 또는 긴 transaction으로 발생한다.

lock wait와 deadlock은 서로 다른 현상이며 deadlock은 DB가 일부 transaction을 중단해 순환 대기를 해소한다.

slow query는 인덱스 누락, 비효율적 계획, 데이터 증가, I/O 병목 또는 통계 변화와 관련될 수 있다.

storage 부족과 높은 queue depth는 쓰기와 checkpoint 성능을 악화시킬 수 있다.

replica lag는 복제 적용 속도가 쓰기 속도를 따라가지 못할 때 증가한다.

---

# 계층 구조

```text
HTTP request
     |
Spring transaction
     |
HikariCP pool
     |
JDBC and network
     |
RDS endpoint
     |
lock / query / memory / storage
     |
Primary and Replica
```

애플리케이션 timeout은 pool 대기, network 연결, query 실행 또는 lock 대기 중 어디에서든 발생할 수 있다.

---

# 증거 기반 진단 순서

## 1. 영향을 확인한다

읽기와 쓰기, 특정 query, tenant, API와 AZ 중 어디에 영향이 집중되는지 확인한다.

데이터 정합성 위험이 있는지 판단하고 쓰기 성공 여부가 불명확한 요청을 별도로 추적한다.

## 2. 최근 변경을 확인한다

배포, schema migration, index 변경, pool 설정, Auto Scaling, DB 변경과 failover event를 확인한다.

## 3. RDS 상태와 메트릭을 확인한다

```bash
aws rds describe-db-instances \
  --db-instance-identifier "$DB_INSTANCE_ID" \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address,Storage:AllocatedStorage,MultiAZ:MultiAZ}'
aws rds describe-events \
  --source-type db-instance \
  --source-identifier "$DB_INSTANCE_ID" \
  --duration 60
```

DBConnections, CPUUtilization, FreeableMemory, FreeStorageSpace, ReadLatency, WriteLatency와 DiskQueueDepth를 같은 시간축에서 본다.

ReplicaLag는 replica별로 확인하고 읽기 일관성 요구와 비교한다.

## 4. Hikari 지표를 확인한다

active, idle, pending, max, connection acquisition time과 timeout을 확인한다.

active가 max에 붙고 pending이 증가하면 pool 대기이며 DB connection 상한과 긴 transaction을 함께 조사한다.

모든 애플리케이션 인스턴스의 `maximum-pool-size` 합과 운영 및 배치 연결을 DB 한도 안에서 계산한다.

## 5. 잠금과 query를 확인한다

엔진의 Performance Insights와 승인된 진단 view에서 top wait, top SQL과 DB load를 확인한다.

긴 transaction, lock holder와 blocked session을 찾아 업무 영향과 종료 위험을 평가한다.

실행 중 session을 무분별하게 종료하면 rollback 부하와 데이터 작업 중단을 만들 수 있으므로 승인 절차를 따른다.

실행 계획은 운영 데이터 분포와 bind 값 특성을 고려해 읽는다.

## 6. 로그를 연결한다

```sql
fields @timestamp, traceId, message, exception
| filter message like /Hikari|JDBC|SQL|timeout|deadlock/
| stats count() by exception, bin(1m)
| sort @timestamp desc
```

애플리케이션 query 식별자와 DB의 top SQL을 연결하되 개인 정보와 SQL parameter는 마스킹한다.

## 7. 가설을 검증한다

연결 고갈 가설은 Hikari pending 증가와 DBConnections 포화가 함께 나타나는지 확인한다.

lock 가설은 DB wait event와 blocked transaction이 실제 느린 API 시간대에 존재하는지 확인한다.

failover 가설은 RDS event, 연결 단절과 endpoint 재해석 시각을 대조한다.

storage 가설은 여유 공간, queue, latency와 write workload를 함께 본다.

## 8. 안전하게 완화한다

비필수 읽기나 batch를 제한해 핵심 transaction의 용량을 확보한다.

문제 query 배포는 검증된 버전으로 되돌리고 replica lag가 크면 stale read 허용 범위에 따라 읽기 경로를 조정한다.

무분별한 즉시 재시도는 connection 폭주와 중복 쓰기를 만들므로 backoff, jitter, 최대 횟수와 멱등성을 갖춘 경우에만 사용한다.

---

# Spring Boot 관찰 포인트

`hikaricp.connections.active`, `idle`, `pending`, `max`와 acquisition timeout을 수집한다.

transaction 경계를 짧게 유지하고 외부 API 호출을 DB transaction 안에 두지 않는다.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 15
      connection-timeout: 3000
      validation-timeout: 1000
  jpa:
    open-in-view: false
```

설정값은 예시이며 실제 한도, 인스턴스 수와 workload를 측정해 결정한다.

failover 후 stale connection을 제거하고 endpoint DNS를 다시 해석하는 동작을 사전 시험한다.

---

# 대응과 복구

연결 고갈은 leak과 긴 transaction을 줄이고 전체 pool 예산을 재설계해 복구한다.

lock과 slow query는 blocker, 실행 계획, index와 transaction 순서를 근거로 수정한다.

failover 뒤 쓰기와 읽기, transaction 재시도 결과 및 데이터 정합성을 확인한다.

복구 후 지연된 batch와 retry가 다시 DB를 포화시키지 않도록 점진적으로 재개한다.

---

# 재발 방지

Hikari pending과 DBConnections를 함께 경보로 관리한다.

schema 변경은 lock 영향과 롤백 계획을 포함해 사전 검증한다.

RTO와 RPO에 맞춰 failover와 point-in-time restore를 정기적으로 연습한다.

slow query와 replica lag의 기준선을 workload 변화에 따라 갱신한다.

---

# 기억해야 할 내용

- DB 장애는 연결, 잠금, query, storage와 가용성 계층으로 나누어 본다.
- Hikari 지표와 RDS 지표를 같은 시간축에서 비교한다.
- 무분별한 재시도는 장애를 증폭하고 중복 쓰기를 만들 수 있다.
- failover 후 endpoint DNS와 기존 연결의 수명을 고려한다.
- 복구에는 데이터 정합성 검증이 포함된다.

---

# 다음 Chapter

다음 장에서는 [Redis 장애](/aws-backend/part-15/06-redis-failure/)를 분석한다.
