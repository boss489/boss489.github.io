---
title: "Chapter 04. TTL"
permalink: /aws-backend/part-06/04-ttl/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. TTL
## 캐시 데이터의 수명

> **학습 목표**
>
> - TTL의 역할을 설명할 수 있다.
> - TTL이 너무 길거나 짧을 때의 문제를 이해한다.
> - 데이터 성격에 따라 TTL을 정하는 기준을 설명할 수 있다.
> - Redis 만료와 eviction의 차이를 구분할 수 있다.

---

# 왜 TTL이 필요한가

상품 상세 캐시에 만료 시간을 설정하지 않은 채 신상품을 계속 추가한다고 가정한다.

판매가 끝난 상품도 메모리에 남고 가격 변경 시 무효화를 한 번 놓치면 오래된 가격이 무기한 노출된다.

메모리가 가득 차면 Redis는 eviction 정책에 따라 기존 키를 제거하거나 새 쓰기를 거부한다.

반대로 모든 상품 TTL을 1초로 설정하면 대부분의 요청이 DB까지 도달하여 캐시를 둔 의미가 사라진다.

TTL은 데이터 신선도, DB 부하, 메모리 사용량 사이의 균형을 수치로 표현하는 운영 정책이다.

---

# TTL이란?

TTL(Time To Live)은 캐시 데이터가 유지되는 시간이다.

Redis에서 `SET key value EX 60`은 키를 60초 뒤 만료되도록 저장하고, `PTTL`은 남은 시간을 밀리초 단위로 확인한다.

만료된 키는 논리적으로 접근할 수 없으며 Redis의 수동적 만료와 주기적인 능동 만료 과정에서 메모리에서 제거된다.

TTL 만료는 키의 수명이 끝난 것이고 eviction은 메모리 한계 때문에 정책에 따라 키를 밀어내는 것이므로 원인이 다르다.

---

# 동작 흐름

```
Client          App            Redis           DB
  │ GET product    │              │             │
  ├───────────────>│              │             │
  │                ├─ GET p:1 ───>│             │
  │                │<── miss ─────┤             │
  │                ├──────── SELECT ────────────>│
  │                │<──────── product ───────────┤
  │                ├─ SET p:1 EX 300 ───────────>│
  │<── product ────┤              │             │
  │       300초 뒤 키가 만료되고 다음 요청은 다시 DB를 조회한다.
```

1. 애플리케이션이 Cache Miss를 확인한다.
2. DB 원본을 조회한다.
3. 데이터 특성에 맞는 TTL을 함께 지정한다.
4. TTL 안의 요청은 캐시에서 처리한다.
5. 만료 후 첫 요청이 다시 캐시를 구성한다.

---

# TTL을 정하는 기준

| 데이터 | 변경 특성 | TTL 방향 | 추가 전략 |
|--------|-----------|----------|-----------|
| 카테고리 목록 | 변경이 드묾 | 길게 | 변경 시 명시적 삭제 |
| 상품 설명 | 가끔 변경 | 중간 | Cache Aside |
| 가격·재고 표시 | 자주 변경 | 짧게 | 이벤트 무효화, 원본 재검증 |
| 존재하지 않는 ID | 반복 공격 가능 | 매우 짧게 | Negative Cache |
| 세션 | 보안 정책과 연동 | 유휴 시간 기준 | 접근 시 만료 연장 검토 |

정답인 TTL 공식은 없으며 허용 가능한 오래된 시간, 변경 빈도, 조회량, 원본 비용을 함께 측정해야 한다.

긴 TTL은 적중률을 높이지만 오래된 값과 메모리 점유를 늘리고, 짧은 TTL은 신선도를 높이지만 Miss와 DB 부하를 늘린다.

---

# Redis eviction policy

`maxmemory`에 도달했을 때 Redis가 어떤 키를 제거할지는 `maxmemory-policy`가 결정한다.

| 정책 | 대상 | 동작 |
|------|------|------|
| `noeviction` | 없음 | 제거하지 않고 메모리가 필요한 쓰기에 오류 반환 |
| `allkeys-lru` | 모든 키 | 최근 사용이 적은 키를 근사 방식으로 제거 |
| `volatile-lru` | TTL이 있는 키 | TTL 키 중 최근 사용이 적은 키 제거 |
| `allkeys-lfu` | 모든 키 | 사용 빈도가 낮은 키 제거 |
| `volatile-ttl` | TTL이 있는 키 | 남은 TTL이 짧은 키를 우선 제거 |
| `allkeys-random` | 모든 키 | 임의의 키 제거 |

`volatile-*` 정책에서 TTL 키가 없으면 제거할 후보가 부족해 쓰기 오류가 발생할 수 있다.

캐시 전용 인스턴스는 `allkeys-lru`나 `allkeys-lfu`를 검토할 수 있지만 세션과 캐시를 같은 클러스터에 섞으면 제거 기준이 서비스 의미와 충돌한다.

ElastiCache 파라미터 그룹에서 변경 가능한 설정과 엔진별 기본값은 운영 중인 엔진 버전의 AWS 문서를 확인해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

캐시 전체 기본 TTL은 `RedisCacheManager`로 설정하고 캐시별 TTL은 초기 구성으로 나눈다.

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    RedisCacheManager redisCacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration defaults = RedisCacheConfiguration
                .defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(5))
                .disableCachingNullValues();

        Map<String, RedisCacheConfiguration> configs = Map.of(
                "categories", defaults.entryTtl(Duration.ofHours(1)),
                "products", defaults.entryTtl(Duration.ofMinutes(5)),
                "stock-view", defaults.entryTtl(Duration.ofSeconds(10)));

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(defaults)
                .withInitialCacheConfigurations(configs)
                .build();
    }
}
```

직접 저장할 때는 TTL 없는 `set`을 실수로 호출하지 않도록 저장 메서드 안에서 수명을 강제한다.

```java
@Component
public class VerificationCodeStore {
    private static final Duration CODE_TTL = Duration.ofMinutes(3);
    private final StringRedisTemplate redisTemplate;

    public VerificationCodeStore(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void save(String requestId, String code) {
        redisTemplate.opsForValue()
                .set("verification:" + requestId, code, CODE_TTL);
    }
}
```

```yaml
spring:
  cache:
    type: redis
  data:
    redis:
      host: ${REDIS_HOST}
      timeout: 1s
```

TTL 지터는 기준 시간에 작은 난수를 더해 많은 키가 같은 순간에 만료되는 현상을 줄인다.

```java
Duration ttlWithJitter(Duration base) {
    long jitter = ThreadLocalRandom.current().nextLong(0, 31);
    return base.plusSeconds(jitter);
}
```

---

# 실무에서는 어떻게 사용할까

초기 TTL은 도메인 허용치로 정하고 운영 지표를 보며 조정한다.

키별 TTL 분포, Hit Ratio, expired keys, evicted keys, 메모리 사용률, DB QPS를 같은 대시보드에서 비교한다.

정확한 변경 반영이 필요하면 TTL만 기다리지 말고 DB 커밋 후 캐시 삭제를 함께 적용한다.

세션, 분산 락, 일반 캐시는 만료 의미와 장애 영향이 다르므로 가능하면 키 공간과 클러스터를 분리한다.

---

# 장애 사례

대량 상품을 배치로 같은 시각에 적재하고 동일 TTL을 부여하면 같은 시각에 동시 Miss가 발생하는 Cache Stampede로 DB가 급증한다.

TTL 지터, 사전 갱신, 짧은 분산 락으로 만료 시점을 분산한다.

TTL을 누락한 임시 키가 누적되면 메모리 고갈 후 eviction이 급증하거나 `noeviction`에서 쓰기 오류가 발생한다.

`volatile-lru`를 사용하면서 영구 키가 대부분이면 제거 가능한 키가 부족하므로 정책과 데이터 구성을 함께 확인해야 한다.

---

# 주의할 점

TTL은 정합성을 보장하는 트랜잭션 장치가 아니라 오래된 값이 남는 최대 시간을 제한하는 장치이다.

Redis 재시작과 복제, Failover 상황의 만료 동작은 엔진과 구성에 영향을 받으므로 장애 테스트가 필요하다.

Sliding TTL은 접근할 때마다 수명이 늘어나 인기 키가 영구히 남을 수 있으므로 의도적으로 사용한다.

너무 큰 단일 키는 만료와 재적재 순간의 CPU와 네트워크 부하를 키운다.

---

# 비용과 성능 고려사항

긴 TTL은 Hit Ratio를 높이지만 더 큰 메모리 노드가 필요할 수 있다.

짧은 TTL은 캐시 메모리를 절약하지만 DB I/O와 애플리케이션 CPU, 네트워크 비용을 늘릴 수 있다.

복제본과 Multi-AZ는 가용성을 높이는 대신 노드 수와 데이터 복제 비용을 증가시킨다.

노드 확장 전에 사용되지 않는 키, 값 크기, 직렬화 방식, TTL 누락, eviction 정책을 먼저 점검한다.

---

# 기억해야 할 내용

- TTL은 캐시 데이터가 유효한 시간을 제한한다.
- 긴 TTL은 적중률을 높이지만 오래된 값과 메모리 점유를 늘린다.
- 짧은 TTL은 신선도를 높이지만 DB 부하를 늘린다.
- 만료와 메모리 부족에 의한 eviction은 서로 다른 현상이다.
- `noeviction`, `allkeys-lru`, `volatile-lru` 등 정책의 대상을 이해해야 한다.
- 같은 TTL이 몰리지 않도록 지터를 적용할 수 있다.
- 가격과 재고처럼 민감한 데이터는 짧게 두거나 캐시하지 않는 선택도 필요하다.

---

# 다음 Chapter

다음 Chapter에서는 여러 애플리케이션 서버가 로그인 상태를 공유하는 **[Redis Session](/aws-backend/part-06/05-redis-session/)** 을 학습한다.

세션 TTL이 사용자 경험과 보안, Redis 장애 범위에 어떤 영향을 주는지 살펴본다.


