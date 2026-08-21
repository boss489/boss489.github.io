---
title: "Chapter 01. Cache Overview"
permalink: /aws-backend/part-06/01-cache-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Cache Overview
## 느린 조회를 빠르게 만드는 중간 저장소

> **학습 목표**
>
> - Cache를 사용하는 이유를 설명할 수 있다.
> - Cache Hit와 Cache Miss의 흐름을 설명할 수 있다.
> - Redis, Memcached, ElastiCache의 역할을 구분할 수 있다.
> - 성능과 데이터 정합성 사이의 비용을 판단할 수 있다.

---

# 왜 Cache가 필요한가

상품 상세 조회 API가 요청마다 상품, 가격, 카테고리 정보를 데이터베이스에서 조합한다고 가정한다.

인기 상품에 초당 수천 건의 조회가 몰리면 같은 결과를 반복해서 계산하느라 DB 커넥션과 CPU가 소진되고 응답 시간도 길어진다.

상품 정보가 1분에 한 번만 바뀐다면 매 요청마다 원본 저장소를 조회할 이유가 없다.

최근 결과를 메모리 기반 저장소에 보관하면 애플리케이션은 대부분의 요청을 DB보다 빠른 경로에서 처리할 수 있다.

캐시는 느린 원본 저장소 앞에 빠른 계층을 두어 **응답 지연과 원본 부하를 함께 줄이는 성능 도구**이다.

다만 캐시를 추가하면 같은 데이터가 두 곳에 존재하므로 오래된 값, 만료, 장애 복구라는 새로운 문제가 생긴다.

---

# Cache란?

Cache는 자주 읽는 데이터를 더 빠른 저장소에 임시로 저장하는 방식이다.

캐시에 데이터가 있으면 Cache Hit, 없으면 Cache Miss라고 한다.

Cache Hit 비율이 높을수록 DB 조회와 네트워크 왕복이 줄어들지만, 적중률만 높이려고 TTL을 무한히 늘리면 데이터 정합성이 나빠질 수 있다.

![Cache patterns](/assets/aws-backend/cache-patterns.png)

| 구분 | 역할 | 대표 저장소 |
|------|------|-------------|
| 원본 데이터 | 영속성과 정합성 보장 | Aurora, RDS |
| 애플리케이션 캐시 | 반복 조회 결과 재사용 | Redis, Memcached |
| 로컬 캐시 | 프로세스 내부의 초고속 조회 | Caffeine |
| CDN 캐시 | 사용자와 가까운 위치에서 콘텐츠 제공 | CloudFront |

---

# 동작 흐름

일반적인 읽기 캐시는 애플리케이션이 캐시를 먼저 확인하고 Miss일 때만 DB에 접근한다.

```
Client          App            Redis           DB
  │ GET /products/1 │              │             │
  ├────────────────>│              │             │
  │                 ├─ GET p:1 ───>│             │
  │                 │<── miss ─────┤             │
  │                 ├──────── SELECT ────────────>│
  │                 │<──────── product ───────────┤
  │                 ├─ SET p:1 EX 60 ────────────>│
  │<──── 200 OK ────┤              │             │
```

1. 클라이언트가 상품 조회를 요청한다.
2. 애플리케이션이 Redis에서 키를 조회한다.
3. Miss이면 DB에서 원본 데이터를 조회한다.
4. 결과를 TTL과 함께 Redis에 저장한다.
5. 다음 요청부터 TTL이 끝날 때까지 캐시 값을 반환한다.

---

# 대표 캐시 패턴 비교

| 패턴 | 읽기 | 쓰기 | 적합한 상황 |
|------|------|------|-------------|
| Cache Aside | Miss일 때 앱이 적재 | DB 변경 후 캐시 삭제 | 일반적인 조회 API |
| Write Through | 주로 캐시 조회 | DB와 캐시를 함께 갱신 | 쓰기 직후 조회가 잦은 데이터 |
| Write Behind | 캐시 조회 | 캐시에 쓴 뒤 비동기 DB 반영 | 높은 쓰기 처리량, 손실 대응 가능 |
| Read Through | 캐시가 원본 조회 담당 | 구현체에 따라 다름 | 캐시 계층을 추상화할 때 |

[Cache Aside](/aws-backend/part-06/02-cache-aside/)는 가장 흔하고 단순하지만 첫 요청은 느리며 무효화 설계가 필요하다.

[Write Through](/aws-backend/part-06/03-write-through/)는 읽기 적중률을 높이는 대신 쓰기 경로의 실패 처리가 복잡하다.

---

# Redis와 Memcached

| 항목 | Redis | Memcached |
|------|-------|-----------|
| 자료 구조 | 문자열, Hash, Set, Stream 등 | 단순 Key-Value |
| 영속성 | 자체 Redis에서 선택 가능 | 제공하지 않음 |
| 복제와 고가용성 | 복제 구조 지원 | 클라이언트 분산 중심 |
| 주요 용도 | 캐시, 세션, 락, 메시징 | 단순 객체 캐시 |

AWS ElastiCache는 Redis OSS, Valkey 또는 Memcached 호환 엔진을 관리형으로 제공하며, 실제 지원 엔진과 버전은 생성 시점의 AWS 문서를 확인해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

Spring Boot 3.x에서는 Spring Cache 추상화와 Spring Data Redis를 조합하면 비즈니스 코드와 저장소 구현을 분리할 수 있다.

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
  cache:
    type: redis
```

`RedisCacheManager`에 기본 TTL을 지정하고 캐시별 정책은 데이터 성격에 맞게 분리한다.

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    RedisCacheManager redisCacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration defaults = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(5))
                .disableCachingNullValues();

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(defaults)
                .build();
    }
}
```

서비스는 저장소 종류를 몰라도 `@Cacheable`, `@CachePut`, `@CacheEvict`로 캐시 생명주기를 표현할 수 있다.

```java
@Service
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Cacheable(cacheNames = "products", key = "#productId")
    public Product findProduct(long productId) {
        return productRepository.findById(productId)
                .orElseThrow(ProductNotFoundException::new);
    }

    @CacheEvict(cacheNames = "products", key = "#productId")
    public void deleteProduct(long productId) {
        productRepository.deleteById(productId);
    }
}
```

세밀한 원자 연산이나 자료 구조가 필요하면 `RedisTemplate` 또는 문자열 중심의 `StringRedisTemplate`을 직접 사용한다.

---

# 실무에서는 어떻게 사용할까

먼저 느린 쿼리와 호출 빈도를 측정하고, 읽기가 많으며 약간의 지연된 갱신을 허용하는 데이터부터 캐시한다.

상품 설명, 카테고리, 권한 메타데이터는 후보가 될 수 있지만 재고 차감과 결제 잔액은 원본 정합성을 우선한다.

키에는 서비스와 버전을 포함한 `catalog:v1:product:123` 같은 규칙을 사용하고 값 크기, TTL, Hit Ratio를 함께 관찰한다.

캐시 장애 시 DB로 우회할 수 있더라도 갑작스러운 전체 우회가 DB를 무너뜨릴 수 있으므로 요청 제한과 점진적 복구가 필요하다.

---

# 장애 사례와 주의할 점

인기 키가 만료되는 순간 수많은 요청이 동시에 DB로 향하는 현상을 **Cache Stampede**라고 한다.

짧은 분산 락, 요청 병합, 사전 갱신, TTL 지터를 사용하면 동시 Miss를 분산할 수 있다.

존재하지 않는 키를 반복 조회하는 Cache Penetration은 짧은 Negative Cache나 입력 검증으로 완화한다.

TTL 없이 키를 계속 쌓으면 메모리가 고갈되고 eviction 또는 쓰기 실패가 발생하므로 모든 임시 데이터에 수명 정책을 둔다.

DB 변경 후 캐시 삭제에 실패하면 오래된 값을 반환하므로 재시도와 모니터링을 설계해야 한다.

---

# 비용과 성능 고려사항

ElastiCache 비용은 주로 노드 타입과 수, 실행 시간, 복제본, Multi-AZ 구성, 백업, 데이터 전송량에 영향을 받는다.

큰 객체는 메모리와 네트워크를 함께 소비하므로 필요한 필드만 저장하고 직렬화 크기를 측정한다.

평균 응답 시간뿐 아니라 p95·p99 지연, Hit Ratio, 메모리 사용률, eviction 수, 연결 수를 관찰해야 한다.

캐시 노드를 키우기 전에 TTL과 키 분포, 압축, 불필요한 데이터부터 점검하는 편이 안전하다.

---

# 기억해야 할 내용

- 캐시는 반복 조회를 빠른 저장소에서 처리해 응답 시간과 DB 부하를 줄인다.
- Cache Hit Ratio와 데이터 신선도는 함께 최적화해야 한다.
- Cache Aside, Write Through 등 패턴마다 읽기와 쓰기의 책임이 다르다.
- Redis는 캐시 외에도 세션, 락, 메시징에 활용할 수 있다.
- TTL과 무효화는 캐시 설계의 필수 요소이다.
- 캐시 장애가 DB 장애로 번지지 않도록 보호 장치를 둬야 한다.
- 정합성이 더 중요한 데이터에는 캐시를 적용하지 않는 선택도 필요하다.

---

# 다음 Chapter

다음 Chapter에서는 가장 널리 사용하는 읽기 전략인 **[Cache Aside](/aws-backend/part-06/02-cache-aside/)** 를 학습한다.

Cache Hit와 Miss의 실제 구현, 변경 시 무효화, Cache Stampede 대응을 Spring Boot 코드로 살펴본다.

