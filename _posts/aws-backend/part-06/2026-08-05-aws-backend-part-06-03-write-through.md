---
title: "Chapter 03. Write Through"
permalink: /aws-backend/part-06/03-write-through/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Write Through
## 저장 시 캐시도 함께 갱신하는 패턴

> **학습 목표**
>
> - Write Through의 흐름을 설명할 수 있다.
> - Cache Aside와 차이를 구분할 수 있다.
> - 쓰기 지연과 장애 처리 이슈를 이해한다.
> - Spring Boot에서 갱신 순서와 실패 정책을 설계할 수 있다.

---

# 왜 Write Through가 필요한가

관리자가 상품 가격을 변경한 직후 대규모 프로모션 트래픽이 시작된다고 가정한다.

Cache Aside에서 캐시를 삭제하면 첫 조회가 DB를 읽어 캐시를 채우므로 인기 상품 여러 개가 동시에 Miss를 일으킬 수 있다.

쓰기 직후 반드시 많은 읽기가 예상된다면 변경 과정에서 최신 값을 캐시에 미리 넣어 첫 조회 지연을 없앨 수 있다.

Write Through는 읽기 성능을 예측 가능하게 만드는 대신 한 번의 쓰기에 DB와 캐시라는 두 저장소를 다루는 복잡성을 추가한다.

---

# Write Through란?

Write Through는 데이터를 저장할 때 DB와 캐시를 함께 갱신하는 방식이다.

엄밀한 캐시 제품의 Write Through는 캐시 계층이 DB 쓰기까지 책임지기도 하지만, 일반적인 Spring 애플리케이션에서는 서비스가 DB 저장과 캐시 갱신을 조정한다.

DB가 원본인 구조에서는 DB 커밋 성공을 확인한 뒤 캐시에 새 값을 넣는 순서가 이해하기 쉽다.

두 저장소를 하나의 로컬 트랜잭션으로 묶을 수 없으므로 완전한 원자성을 가정해서는 안 된다.

---

# 동작 흐름

```
Client          App             DB            Redis
  │ PUT product   │              │               │
  ├──────────────>│              │               │
  │               ├─ UPDATE ────>│               │
  │               │<─ commit ────┤               │
  │               ├──────────── SET product:1 ──>│
  │               │<──────────── OK ─────────────┤
  │<── 200 OK ────┤              │               │
```

1. 애플리케이션이 요청을 검증하고 DB 데이터를 변경한다.
2. DB 트랜잭션을 커밋하여 원본을 확정한다.
3. 확정된 최신 표현을 TTL과 함께 캐시에 저장한다.
4. 이후 읽기는 캐시에서 최신 값에 가깝게 처리된다.
5. 캐시 갱신이 실패하면 정책에 따라 재시도하거나 키를 삭제한다.

DB보다 캐시를 먼저 갱신하면 DB 저장 실패 시 존재하지 않는 상태가 캐시에 노출될 수 있으므로 보통 피한다.

---

# 패턴 비교

| 항목 | Cache Aside | Write Through | Write Behind |
|------|-------------|---------------|--------------|
| 쓰기 처리 | DB 후 캐시 삭제 | DB 후 캐시 갱신 | 캐시 후 비동기 DB 반영 |
| 쓰기 지연 | 낮음 | 상대적으로 높음 | 낮음 |
| 쓰기 직후 읽기 | 첫 요청 Miss | Hit 가능 | Hit 가능 |
| 실패 위험 | 삭제 실패로 오래된 값 | 이중 쓰기 불일치 | 유실·순서 역전 |
| 구현 난이도 | 낮음 | 중간 | 높음 |

단순한 조회 시스템에서는 [Cache Aside](/aws-backend/part-06/02-cache-aside/)가 더 적은 운영 비용으로 충분할 수 있다.

Write Behind는 쓰기 처리량이 높지만 캐시 장애 전에 DB 반영이 끝나지 않으면 데이터가 유실될 수 있어 별도의 내구성 설계가 필요하다.

---

# Spring Boot에서는 어떻게 쓰는가

`@CachePut`은 메서드를 항상 실행하고 반환값으로 캐시를 갱신하므로 Write Through 형태에 사용할 수 있다.

```java
@Service
public class ProductCommandService {
    private final ProductRepository productRepository;

    public ProductCommandService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional
    @CachePut(cacheNames = "products", key = "#productId")
    public ProductResponse changePrice(long productId, BigDecimal price) {
        Product product = productRepository.findById(productId)
                .orElseThrow(ProductNotFoundException::new);
        product.changePrice(price);
        return ProductResponse.from(product);
    }
}
```

트랜잭션이 최종 커밋되기 전에 캐시가 갱신되는 문제를 피하려면 트랜잭션 완료 후 이벤트를 처리할 수 있다.

```java
public record ProductChangedEvent(long productId) {
}
```

```java
@Component
public class ProductCacheUpdater {
    private final ProductRepository productRepository;
    private final RedisTemplate<String, ProductResponse> redisTemplate;

    public ProductCacheUpdater(
            ProductRepository productRepository,
            RedisTemplate<String, ProductResponse> redisTemplate) {
        this.productRepository = productRepository;
        this.redisTemplate = redisTemplate;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void update(ProductChangedEvent event) {
        productRepository.findById(event.productId())
                .map(ProductResponse::from)
                .ifPresent(value -> redisTemplate.opsForValue().set(
                        "catalog:v1:product:" + event.productId(),
                        value,
                        Duration.ofMinutes(5)));
    }
}
```

프로세스가 DB 커밋 직후 종료되면 인메모리 이벤트는 사라질 수 있으므로 강한 복구가 필요하면 Transactional Outbox와 재처리를 사용한다.

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      timeout: 1s
```

캐시 호출 타임아웃은 DB 트랜잭션과 사용자 요청을 오래 붙잡지 않도록 짧게 설정하되 실제 네트워크 지연을 측정해 결정한다.

---

# 실무에서는 어떻게 사용할까

쓰기 직후 읽기 빈도가 높은 상품 정보, 사용자 설정, 권한 메타데이터가 후보가 된다.

매우 드물게 읽는 데이터까지 매번 캐시에 쓰면 메모리와 네트워크만 낭비하므로 접근 패턴을 먼저 측정한다.

캐시 갱신 실패를 사용자 요청 실패로 돌릴지, 성공으로 응답하고 비동기로 복구할지는 데이터 중요도에 따라 결정한다.

대부분의 일반 캐시는 DB 성공 후 캐시 실패를 허용하고 키 삭제 또는 재시도로 수렴시키는 편이 가용성에 유리하다.

---

# 장애 사례

DB 저장은 성공했지만 Redis 타임아웃이 발생하면 사용자는 성공한 변경 뒤에도 이전 캐시 값을 볼 수 있다.

이 경우 기존 키를 삭제하고 다음 조회가 DB에서 다시 적재하게 하거나 Outbox 이벤트를 반복 처리한다.

동시에 발생한 두 변경의 캐시 갱신 순서가 뒤집히면 더 오래된 값이 마지막에 저장될 수 있다.

버전 번호나 `updatedAt`을 값에 포함하고 Lua 스크립트로 더 최신 버전만 반영하면 순서 역전을 줄일 수 있다.

재시도가 중복 실행되어도 같은 결과가 되도록 캐시 갱신 작업을 멱등하게 설계한다.

---

# 주의할 점

DB와 Redis에 대한 분산 트랜잭션이 자동으로 제공된다고 생각하면 안 된다.

캐시 갱신 때문에 DB 트랜잭션을 오래 열면 락 보유 시간과 커넥션 사용량이 증가한다.

TTL을 함께 지정하지 않으면 사용되지 않는 데이터가 계속 남을 수 있다.

직렬화 포맷이 바뀌는 배포에서는 키 버전을 변경하거나 하위 호환성을 유지한다.

---

# 비용과 성능 고려사항

모든 쓰기가 Redis 네트워크 호출을 추가하므로 쓰기 지연과 연결 수가 증가한다.

읽히지 않을 값도 캐시에 들어가는 Cache Pollution이 발생하면 더 큰 노드와 eviction 증가로 이어진다.

ElastiCache의 노드 수와 메모리, 복제본, Multi-AZ, 데이터 전송이 주요 비용 요소이다.

읽기 절감량이 추가 쓰기 비용보다 큰지 Hit Ratio, 쓰기 처리량, p99 응답 시간을 기준으로 판단한다.

파이프라이닝은 여러 독립 명령의 왕복을 줄일 수 있지만 원자성을 보장하는 기능은 아니다.

---

# 기억해야 할 내용

- Write Through는 저장할 때 DB와 캐시를 함께 갱신한다.
- 쓰기 직후 캐시 Hit를 만들지만 쓰기 지연과 실패 지점이 늘어난다.
- DB를 원본으로 삼고 커밋 후 캐시를 갱신하는 순서가 일반적이다.
- `@CachePut`은 메서드 결과로 캐시를 갱신한다.
- 두 저장소를 하나의 로컬 트랜잭션처럼 취급해서는 안 된다.
- 캐시 실패는 삭제, 재시도, Outbox로 최종 수렴시킨다.
- 단순한 서비스는 Cache Aside가 더 적합할 수 있다.

---

# 다음 Chapter

다음 Chapter에서는 캐시 데이터의 수명을 정하는 **[TTL](/aws-backend/part-06/04-ttl/)** 을 학습한다.

만료 시간이 성능과 정합성, 메모리 사용량, eviction 정책에 어떤 영향을 주는지 살펴본다.

