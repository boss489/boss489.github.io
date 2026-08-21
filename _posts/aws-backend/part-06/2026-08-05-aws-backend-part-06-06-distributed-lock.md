---
title: "Chapter 06. Distributed Lock"
permalink: /aws-backend/part-06/06-distributed-lock/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Distributed Lock
## 여러 서버 사이의 동시 실행 제어

> **학습 목표**
>
> - Distributed Lock이 필요한 상황을 설명할 수 있다.
> - Redis Lock 사용 시 TTL이 왜 중요한지 이해한다.
> - Lock이 정답이 아닌 경우를 구분할 수 있다.
> - 소유자 토큰으로 락을 안전하게 해제할 수 있다.

---

# 왜 Distributed Lock이 필요한가

선착순 쿠폰이 한 장 남았을 때 두 사용자의 요청이 서로 다른 애플리케이션 서버에 동시에 도착한다고 가정한다.

각 서버의 `synchronized`는 자기 JVM 안에서만 동작하므로 두 서버가 모두 남은 수량을 1로 읽고 쿠폰을 발급할 수 있다.

재고 차감, 정산 배치, 외부 API 중복 호출처럼 하나의 논리 작업을 한 실행자만 수행해야 할 때 프로세스 밖의 조정 수단이 필요하다.

Distributed Lock은 여러 서버가 공유하는 저장소를 이용해 임계 구역에 동시에 들어가는 실행자 수를 제한한다.

하지만 락은 네트워크 지연과 장애 상태를 추가하므로 DB의 유니크 제약이나 원자적 UPDATE로 해결할 수 있다면 그 방법이 더 단순하다.

---

# Distributed Lock이란?

분산 락은 여러 프로세스가 동일한 자원에 접근할 때 한 소유자만 일정 시간 동안 작업 권한을 갖게 하는 동시성 제어 기법이다.

락 키는 `lock:coupon:42`처럼 보호할 자원을 식별하고 값에는 요청마다 생성한 고유한 소유자 토큰을 저장한다.

만료 시간은 소유 서버가 종료되어도 락이 영원히 남는 것을 막지만, 작업보다 짧으면 다른 실행자가 들어와 이중 실행될 수 있다.

---

# 동작 흐름

```
Client A       App A            Redis           App B       Client B
   │             │                │               │            │
   ├─ issue ────>│                │               │<── issue ───┤
   │             ├─ SET NX PX ───>│               │            │
   │             │<──── OK ───────┤<── SET NX PX ─┤            │
   │             │  critical work │──── null ────>│            │
   │             ├─ Lua release ─>│               │            │
   │             │<──── 1 ────────┤               │            │
   │<── success ─┤                │               ├─ retry ───>│
```

1. App A가 임의의 소유자 토큰을 생성한다.
2. `SET key value NX PX <ms>`를 한 명령으로 실행해 키가 없을 때만 TTL과 함께 락을 획득한다.
3. App B는 같은 키 획득에 실패하여 즉시 종료하거나 제한된 시간 동안 재시도한다.
4. App A가 임계 작업을 완료한다.
5. App A는 현재 값이 자기 토큰과 같을 때만 Lua 스크립트로 락을 삭제한다.

---

# 안전한 획득과 해제

락 획득은 `SET lock:coupon:42 <token> NX PX 5000`처럼 조건과 만료를 단일 원자 명령으로 설정해야 한다.

`SETNX` 후 별도로 `PEXPIRE`를 호출하면 두 명령 사이에 프로세스가 종료되어 TTL 없는 락이 남을 수 있다.

단순 `DEL`은 자신의 락이 만료된 뒤 다른 요청이 획득한 새 락까지 삭제할 수 있다.

```text
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

위 Lua 스크립트는 토큰 비교와 삭제를 Redis에서 원자적으로 실행한다.

작업 시간이 예측하기 어렵다면 Redisson의 watchdog 같은 만료 연장 기능을 사용할 수 있지만 프로세스 정지와 네트워크 단절까지 없애는 것은 아니다.

---

# 동시성 제어 방식 비교

| 방식 | 범위 | 장점 | 적합한 상황 |
|------|------|------|-------------|
| `synchronized` | 단일 JVM | 단순하고 빠름 | 서버 한 대의 메모리 상태 |
| DB 유니크 제약 | 데이터 무결성 | 최종 방어선, 단순함 | 쿠폰 중복 발급 방지 |
| 원자적 UPDATE | 단일 DB 행 | 조건과 변경을 한 번에 처리 | `stock > 0` 재고 차감 |
| 비관적 락 | DB 트랜잭션 | 강한 직렬화 | 짧은 DB 작업 |
| Redis 분산 락 | 여러 프로세스 | DB 밖 작업도 조정 | 외부 API, 배치 단일 실행 |

재고는 `UPDATE product SET stock = stock - 1 WHERE id = ? AND stock > 0`의 영향 행 수로 성공을 판단하면 별도 Redis 락보다 단순할 수 있다.

---

# Spring Boot에서는 어떻게 쓰는가

Redisson의 `RLock`은 소유권 확인과 재진입, 대기 시간, lease time 같은 기능을 제공하여 직접 구현 오류를 줄인다.

```java
@Service
public class CouponIssueService {
    private final RedissonClient redissonClient;
    private final CouponTransactionService couponTransactionService;

    public CouponIssueService(
            RedissonClient redissonClient,
            CouponTransactionService couponTransactionService) {
        this.redissonClient = redissonClient;
        this.couponTransactionService = couponTransactionService;
    }

    public void issue(long couponId, long memberId) {
        RLock lock = redissonClient.getLock("lock:coupon:" + couponId);
        boolean acquired = false;

        try {
            acquired = lock.tryLock(200, 3_000, TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new CouponBusyException();
            }
            couponTransactionService.issue(couponId, memberId);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new CouponBusyException(exception);
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

}
```

Spring 프록시 기반 `@Transactional`은 같은 클래스의 private 메서드 직접 호출에 적용되지 않으므로 실제 구현에서는 트랜잭션 서비스를 별도 Bean으로 분리한다.

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      timeout: 500ms
```

lease time을 직접 지정하면 watchdog 자동 연장 동작이 달라질 수 있으므로 사용 중인 Redisson 버전의 계약을 확인한다.

---

# 실무에서는 어떻게 사용할까

락 키의 범위를 전체 작업이 아니라 충돌하는 자원 단위로 좁혀 병렬성을 유지한다.

획득 대기 시간과 작업 제한 시간을 정하고 실패 시 무한 재시도 대신 409 또는 429 응답, 큐잉, 지수 백오프를 선택한다.

락이 있어도 데이터베이스 유니크 제약을 최종 방어선으로 남겨 이중 실행이 데이터 무결성 위반으로 이어지지 않게 한다.

락 획득 실패율, 대기 시간, 보유 시간, 만료 후 작업 지속 횟수를 메트릭으로 수집한다.

---

# 장애 사례

작업이 5초 걸리는데 TTL을 3초로 설정하면 첫 작업이 끝나기 전에 락이 만료되어 두 서버가 동시에 실행한다.

긴 GC Pause나 네트워크 단절로 프로세스가 멈추어도 같은 문제가 발생할 수 있으므로 임계 작업은 멱등하게 만들고 필요하면 fencing token을 검토한다.

락을 획득한 Redis 노드의 쓰기가 복제본에 전달되기 전에 장애가 나고 복제본이 승격되면 새 Primary에 락 정보가 없을 수 있다.

따라서 Redis 복제와 Failover만으로 모든 상황의 상호 배제가 보장된다고 단정해서는 안 된다.

---

# 주의할 점

Redlock은 여러 독립 Redis 노드의 다수결로 락을 얻는 알고리즘이지만 시간 가정과 장애 모델을 둘러싼 논쟁이 있다.

강한 정확성이 필요한 시스템은 Redlock을 무조건 정답으로 채택하지 말고 DB 제약, 합의 시스템, fencing token을 포함해 요구 수준을 검토한다.

락을 잡은 상태로 느린 외부 API를 호출하면 보유 시간이 길어지고 전체 처리량이 급감한다.

락 해제 실패를 무시해도 TTL로 언젠가 풀리지만 그동안 요청이 막히므로 경고와 메트릭을 남긴다.

---

# 비용과 성능 고려사항

락마다 Redis 왕복과 경쟁 대기가 추가되어 처리량과 p99 지연에 영향을 준다.

단일 인기 키에 요청이 몰리면 Redis 노드보다 임계 구역 자체가 병목이므로 락 범위 축소나 큐 기반 직렬화를 검토한다.

ElastiCache 비용은 노드 수와 타입, 복제본, Multi-AZ, 데이터 전송에 영향을 받지만 정확성 요구 때문에 무조건 저렴한 단일 노드를 선택해서는 안 된다.

재시도 폭주는 추가 부하를 만들므로 작은 랜덤 지연과 최대 시도 횟수를 둔다.

---

# 기억해야 할 내용

- 분산 락은 여러 프로세스 사이의 임계 구역 진입을 조정한다.
- 락 획득은 `SET key value NX PX <ms>`처럼 TTL과 조건을 원자적으로 설정한다.
- 락 값에는 요청마다 다른 소유자 토큰을 저장한다.
- 해제는 토큰 비교와 `DEL`을 Lua 스크립트로 원자 실행한다.
- TTL보다 작업이 길어지면 이중 실행이 가능하므로 멱등성을 확보한다.
- Redlock에는 장애 모델과 시간 가정을 둘러싼 논쟁이 있다.
- DB 유니크 제약이나 원자적 UPDATE로 풀 수 있으면 그 방법이 더 단순하다.

---

# 다음 Chapter

다음 Chapter에서는 Redis의 메시징 기능과 AWS 관리형 캐시인 **[Pub/Sub and ElastiCache](/aws-backend/part-06/07-pubsub-elasticache/)** 를 학습한다.

메시지 유실 특성과 ElastiCache의 고가용성, Failover 운영 포인트를 살펴본다.


