---
title: "Chapter 06. Redis Failure"
permalink: /aws-backend/part-15/06-redis-failure/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Redis Failure
## 메모리, 명령과 failover 진단

> **학습 목표**
>
> - Redis가 cache인지 원본 데이터 저장소인지 먼저 구분할 수 있다.
> - memory, eviction, connection, hot key와 slow command를 진단할 수 있다.
> - cache stampede와 failover를 안전하게 완화할 수 있다.

---

# 실제 장애 징후

cache hit ratio가 급락하고 RDS 부하와 API 지연이 동시에 증가할 수 있다.

연결 timeout, `MOVED` 또는 `READONLY` 오류와 failover event가 나타날 수 있다.

eviction 증가, memory 부족, command latency와 특정 shard의 CPU 편중이 관찰될 수 있다.

동일 키 만료 직후 많은 요청이 원본 저장소로 몰리면 cache stampede가 발생한다.

---

# 정의와 가능한 원인

Redis 장애는 Redis 접근 실패뿐 아니라 cache 효율 저하가 원본 시스템을 과부하시키는 상태를 포함한다.

memory 부족은 큰 value, TTL 누락, key 증가와 fragmentation으로 발생할 수 있다.

eviction은 설정된 maxmemory와 eviction policy에 따라 키를 제거하는 동작이며 그 자체가 항상 오류는 아니다.

connection 고갈은 client pool 과대 설정, leak, network 단절 또는 failover 후 stale connection으로 발생한다.

hot key는 요청이 한 key와 shard에 집중되어 전체 cluster 여유와 무관하게 병목을 만든다.

slow command, 큰 collection 조회와 blocking operation은 단일 실행 경로의 지연을 확산시킬 수 있다.

cache stampede는 인기 key가 동시에 만료되거나 cache가 비워질 때 원본 조회가 폭증하는 현상이다.

---

# 계층 구조

```text
Spring Boot
   |
Redis client pool
   |
Configuration endpoint
   |
Primary / Replica / Shard
   |
memory / key / command
   |
source of truth such as RDS
```

Redis가 원본이 아니라 cache라면 Redis 복구뿐 아니라 원본 시스템 보호가 최우선이다.

---

# 증거 기반 진단 순서

## 1. 데이터 역할과 영향을 확인한다

Redis 데이터가 재생성 가능한 cache인지 session, lock 또는 원본 데이터인지 확인한다.

cache miss 증가가 어떤 RDS query와 외부 호출을 증폭시키는지 확인한다.

## 2. 최근 변경을 확인한다

TTL, serializer, key schema, client pool, topology, node 변경과 배포 시각을 확인한다.

## 3. ElastiCache 상태와 메트릭을 확인한다

EngineCPUUtilization, DatabaseMemoryUsagePercentage, FreeableMemory, Evictions, CurrConnections와 NewConnections를 확인한다.

CacheHits, CacheMisses, command latency, replication lag와 network bytes를 shard별로 비교한다.

```bash
aws elasticache describe-cache-clusters \
  --show-cache-node-info \
  --cache-cluster-id "$CACHE_CLUSTER_ID"
aws elasticache describe-events \
  --source-type cache-cluster \
  --duration 60
```

cluster mode에서는 전체 평균보다 node와 shard 편차를 우선 확인한다.

## 4. 안전한 진단 명령을 사용한다

```bash
redis-cli -h "$REDIS_HOST" -p 6379 INFO memory
redis-cli -h "$REDIS_HOST" -p 6379 INFO stats
redis-cli -h "$REDIS_HOST" -p 6379 INFO clients
redis-cli -h "$REDIS_HOST" -p 6379 SLOWLOG GET 20
```

운영에서 전체 key 공간을 막는 명령은 사용하지 않고 sampled metric과 `SCAN` 기반 승인 도구를 사용한다.

SLOWLOG는 command 실행 시간 중심이므로 network 대기 전체를 나타내지 않는다는 점을 고려한다.

## 5. 가설을 검증한다

memory 가설은 used memory, fragmentation, key 증가, eviction과 TTL 분포를 함께 본다.

connection 가설은 client pool pending, CurrConnections, NewConnections와 connection error를 연결한다.

hot key 가설은 특정 shard CPU와 command latency 편중 및 sampled key 접근량으로 검증한다.

slow command 가설은 SLOWLOG와 애플리케이션 trace의 Redis span을 대조한다.

failover 가설은 ElastiCache event, 역할 변경과 client reconnect 시각을 비교한다.

stampede 가설은 특정 key 만료 직후 cache miss와 원본 query가 동시에 증가하는지 확인한다.

## 6. 안전하게 완화한다

cache 장애이면 rate limit, stale cache 허용과 request coalescing으로 원본 저장소를 보호한다.

hot key는 안전하게 복제하거나 key 분할을 검토하되 일관성 요구를 먼저 확인한다.

문제 배포는 롤백하고 cache를 한 번에 비우지 않는다.

failover 중에는 제한된 backoff와 jitter로 재연결하며 무한 재시도를 금지한다.

---

# Spring Boot 관찰 포인트

Lettuce 또는 Jedis의 active, idle, pending connection과 command timeout을 수집한다.

Micrometer의 cache hit, miss, put, eviction 지표를 business endpoint와 연결한다.

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
      timeout: 2s
```

timeout 값은 예시이며 사용자 지연 예산과 network 특성에 맞춰 검증한다.

cache-aside에서는 null cache, TTL jitter와 single-flight를 적용해 반복 miss를 줄일 수 있다.

분산 lock 장애에서는 lease 만료와 fencing 없이 무조건 재획득하면 동시 실행이 발생할 수 있다.

---

# 대응과 복구

memory 부족은 key 크기와 TTL 정책을 수정하고 검증된 방식으로 용량을 조정한다.

connection 문제는 전체 인스턴스의 pool 예산과 reconnect 폭주를 함께 바로잡는다.

복구 후 cache를 점진적으로 warm-up하고 RDS 부하와 hit ratio를 동시에 관찰한다.

Redis가 원본 역할이면 durability 설정과 백업에서 복구한 데이터의 정합성을 별도로 검증한다.

---

# 재발 방지

eviction, memory, connection, latency와 cache miss 증가에 경보를 구성한다.

key 이름 규칙, 최대 value 크기, TTL과 serialization 변경을 리뷰한다.

대규모 동시 만료를 피하도록 TTL jitter와 요청 병합을 설계한다.

failover와 cold cache 상황에서 원본 저장소가 견디는지 부하 시험한다.

---

# 기억해야 할 내용

- Redis가 cache인지 원본인지가 복구 전략을 결정한다.
- eviction은 policy 결과이므로 memory와 hit ratio를 함께 해석한다.
- 평균보다 shard별 hot key와 부하 편차를 확인한다.
- cache stampede에서는 원본 저장소 보호가 우선이다.
- 전체 cache 삭제는 안전한 첫 대응이 아니다.

---

# 다음 Chapter

다음 장에서는 [DNS 장애](/aws-backend/part-15/07-dns-failure/)를 분석한다.
