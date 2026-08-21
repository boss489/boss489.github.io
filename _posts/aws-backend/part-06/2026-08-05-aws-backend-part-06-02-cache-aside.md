---
title: "Chapter 02. Cache Aside"
permalink: /aws-backend/part-06/02-cache-aside/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Cache Aside
## 애플리케이션이 캐시를 직접 채우는 패턴

> **학습 목표**
>
> - Cache Aside 흐름을 설명할 수 있다.
> - Cache Miss 상황에서 DB를 조회하는 과정을 이해한다.
> - 캐시 무효화가 필요한 이유를 설명할 수 있다.
> - Cache Stampede 대응 방법을 선택할 수 있다.

---

# 왜 Cache Aside가 필요한가

상품 상세 API가 하루에 한 번도 조회되지 않는 수백만 개 상품을 모두 캐시에 미리 올리면 메모리 대부분이 낭비된다.

반대로 인기 상품은 같은 SELECT가 반복되어 DB 부하와 응답 지연을 만든다.

실제로 요청된 데이터만 캐시에 올리고 캐시가 비어 있어도 DB에서 정상 조회하려면 Cache Aside가 적합하다.

이 패턴은 애플리케이션이 캐시 사용 여부를 결정하므로 도입과 제거가 쉽고 장애 시 원본 저장소로 우회할 수 있다.

다만 애플리케이션이 조회, 적재, 무효화를 모두 책임지기 때문에 실패 순서와 동시성을 명확히 설계해야 한다.

---

# Cache Aside란?

Cache Aside는 애플리케이션이 캐시를 먼저 조회하고, 값이 없을 때 DB를 조회한 뒤 결과를 캐시에 적재하는 패턴이다.

Lazy Loading이라고도 하며 실제 요청을 받은 데이터만 저장하므로 메모리 효율이 좋다.

캐시는 원본이 아니므로 캐시가 사라져도 DB에서 다시 구성할 수 있어야 한다.

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
  │                ├─ SET p:1 EX 60 ────────────>│
  │<── product ────┤              │             │
```

1. 애플리케이션이 캐시 키 `product:1`을 조회한다.
2. 값이 있으면 역직렬화하여 즉시 반환한다.
3. 값이 없으면 DB에서 상품을 조회한다.
4. 조회 결과를 제한된 TTL과 함께 캐시에 저장한다.
5. DB에도 없다면 예외를 반환하거나 짧은 Negative Cache를 저장한다.

---

# 쓰기와 무효화 흐름

DB를 먼저 갱신한 뒤 캐시를 삭제하는 방식이 일반적이다.

```
Client          App             DB            Redis
  │ PUT product   │              │               │
  ├──────────────>│              │               │
  │               ├─ UPDATE ────>│               │
  │               │<─ commit ────┤               │
  │               ├──────────── DEL product:1 ──>│
  │<── 200 OK ────┤              │               │
```

캐시를 먼저 삭제하고 DB 갱신이 늦어지면 다른 요청이 변경 전 DB 값을 다시 캐시에 넣을 수 있다.

---

# 캐시 패턴 비교

| 항목 | Cache Aside | Write Through | Write Behind |
|------|-------------|---------------|--------------|
| 적재 시점 | 읽기 Miss | 쓰기 시점 | 쓰기 시점 |
| DB 반영 | 앱이 직접 즉시 | 앱 또는 캐시 계층이 즉시 | 비동기 |
| 장점 | 단순하고 메모리 효율적 | 읽기 직후 Hit | 쓰기 처리량 높음 |
| 위험 | 첫 요청 지연, 오래된 값 | 이중 쓰기 실패 | 데이터 손실 가능성 |
| 적합한 경우 | 읽기 중심 API | 쓰기 후 조회가 잦음 | 손실 보완 가능한 대량 쓰기 |

---

# Spring Boot에서는 어떻게 쓰는가

Spring Cache의 `@Cacheable`은 Cache Aside의 조회 분기를 선언적으로 구현한다.

```java
@Service
public class ProductQueryService {
    private final ProductRepository productRepository;

    public ProductQueryService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Cacheable(cacheNames = "products", key = "#productId",
            unless = "#result == null")
    public ProductResponse findById(long productId) {
        return productRepository.findById(productId)
                .map(ProductResponse::from)
                .orElseThrow(ProductNotFoundException::new);
    }
}
```

`@Cacheable`은 프록시 기반이므로 같은 객체 내부에서 메서드를 직접 호출하면 캐시가 적용되지 않는 self-invocation 문제가 있다.

변경 경로에서는 트랜잭션이 성공한 뒤 `@CacheEvict`가 실행되도록 서비스 경계를 구성한다.

```java
@Service
public class ProductCommandService {
    private final ProductRepository productRepository;

    public ProductCommandService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional
    @CacheEvict(cacheNames = "products", key = "#productId")
    public void changeName(long productId, String name) {
        Product product = productRepository.findById(productId)
                .orElseThrow(ProductNotFoundException::new);
        product.changeName(name);
    }
}
```

직접 제어가 필요하면 `StringRedisTemplate`로 키, TTL, 직렬화를 명시한다.

```java
@Service
public class ProductCache {
    private final StringRedisTemplate redisTemplate;
    public ProductCache(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void evict(long productId) {
        redisTemplate.delete("catalog:v1:product:" + productId);
    }
}
```

```yaml
spring:
  cache:
    redis:
      time-to-live: 5m
      cache-null-values: false
```

---

# 실무에서는 어떻게 사용할까

조회량이 많고 변경 빈도가 낮으며 잠시 오래된 값이 보여도 되는 데이터에 우선 적용한다.

키 이름에는 도메인과 스키마 버전을 포함하고 캐시 값은 API 응답 전체보다 재사용 가능한 데이터 구조로 저장한다.

운영에서는 Hit Ratio뿐 아니라 Miss 후 DB 지연, 적재 실패, eviction, 값 크기를 함께 측정한다.

---

# 장애 사례

인기 상품 키가 만료되는 순간 요청 수백 개가 동시에 DB를 조회하면 Cache Stampede가 발생한다.

한 요청만 DB를 조회하도록 짧은 락을 사용하거나, 논리적 만료 후 백그라운드 갱신, TTL 지터를 적용할 수 있다.

Redis 장애 때 모든 요청을 무조건 DB로 우회하면 DB까지 장애가 전파되므로 회로 차단, 요청 제한, 제한된 fallback이 필요하다.

존재하지 않는 상품 ID를 반복 요청하면 매번 DB까지 도달하므로 짧은 Negative Cache 또는 Bloom Filter를 검토한다.

---

# 주의할 점

캐시 값은 권위 있는 데이터가 아니며 정합성 판단과 최종 쓰기는 DB를 기준으로 해야 한다.

긴 TTL만 믿으면 변경 직후 오래된 값이 노출되므로 명시적 삭제를 함께 사용한다.

분산 환경에서 로컬 락은 다른 인스턴스의 동시 Miss를 막지 못한다.

---

# 비용과 성능 고려사항

필요한 데이터만 적재하므로 선적재 방식보다 메모리 비용을 줄일 수 있지만, 낮은 적중률에서는 캐시 비용과 DB 부하가 모두 발생한다.

ElastiCache 비용은 노드 메모리와 수, 복제본, Multi-AZ, 백업, 데이터 전송과 같은 구성에 따라 달라진다.

JSON 직렬화는 편리하지만 값이 커질 수 있으므로 평균·최대 객체 크기와 네트워크 시간을 측정한다.

---

# 기억해야 할 내용

- Cache Aside는 캐시를 먼저 읽고 Miss이면 DB에서 적재한다.
- 실제 요청된 데이터만 저장하므로 메모리 효율이 좋다.
- DB 변경 후 캐시 삭제를 기본으로 하되 실패 복구를 설계한다.
- `@Cacheable`과 `@CacheEvict`로 Spring Cache 패턴을 구현할 수 있다.
- Cache Stampede에는 짧은 락, 사전 갱신, TTL 지터를 적용한다.
- 캐시 장애의 DB 전파를 요청 제한과 회로 차단으로 막아야 한다.
- 캐시는 원본 데이터 저장소가 아니다.

---

# 다음 Chapter

다음 Chapter에서는 저장 시점에 DB와 캐시를 함께 갱신하는 **[Write Through](/aws-backend/part-06/03-write-through/)** 를 학습한다.

읽기 적중률을 높이는 대가로 생기는 이중 쓰기와 실패 복구 문제를 살펴본다.

---

# 주의할 점

데이터 변경 시 캐시를 삭제하거나 갱신해야 한다.

삭제를 놓치면 사용자는 오래된 데이터를 볼 수 있다.

---

# 기억해야 할 내용

Cache Aside는 가장 흔한 읽기 캐시 패턴이다.

쓰기 경로에서 캐시 무효화를 함께 설계해야 한다.


