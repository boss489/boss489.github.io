---
title: "Chapter 07. Pub/Sub and ElastiCache"
permalink: /aws-backend/part-06/07-pubsub-elasticache/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Pub/Sub and ElastiCache
## Redis 메시징과 AWS 관리형 Redis

> **학습 목표**
>
> - Redis Pub/Sub의 기본 구조를 설명할 수 있다.
> - ElastiCache의 역할을 이해한다.
> - 운영 Redis에서 고려할 장애 지점을 설명할 수 있다.
> - Cluster Mode와 고가용성 구성을 구분할 수 있다.

---

# 왜 Pub/Sub과 ElastiCache가 필요한가

상품 가격이 변경되었을 때 여러 Spring Boot 인스턴스의 로컬 캐시를 동시에 지워야 한다고 가정한다.

각 서버를 직접 호출하면 인스턴스 수와 주소를 알아야 하고 한 서버를 놓치면 오래된 가격이 남는다.

발행자가 `product.changed` 채널에 이벤트를 보내고 모든 서버가 구독하면 느슨하게 캐시 무효화를 전달할 수 있다.

직접 Redis를 설치하면 패치, 장애 감지, 복제본 승격, 백업, 용량 확장을 운영해야 한다.

ElastiCache는 이러한 운영 작업의 일부를 AWS가 관리하지만 메시지 의미, TTL, 재연결, 장애 시 사용자 경험은 애플리케이션 책임이다.

---

# Redis Pub/Sub이란?

Pub/Sub은 발행자가 채널에 메시지를 보내면 현재 구독 중인 소비자에게 즉시 전달하는 메시징 모델이다.

발행자는 구독자 목록을 알 필요가 없고 구독자는 같은 메시지를 각자 받을 수 있다.

Redis Pub/Sub 메시지는 저장되지 않으므로 구독자가 끊겨 있거나 처리 중 실패하면 나중에 다시 받을 수 없다.

재처리와 소비 확인이 필요한 주문·결제 이벤트에는 Redis Streams, SQS, SNS, Kafka 같은 내구성 있는 대안을 검토한다.

---

# 동작 흐름

```
Admin          App A           Redis           App B          App C
  │ change price │               │               │              │
  ├─────────────>│               │               │              │
  │              ├─ PUBLISH ────>│               │              │
  │              │               ├─ message ────>│              │
  │              │               ├─ message ──────────────────>│
  │              │               │               ├─ evict local │
  │              │               │               │              ├─ evict
  │<── success ──┤               │               │              │
```

1. App A가 DB의 상품 가격을 변경한다.
2. 커밋 후 상품 ID를 채널에 발행한다.
3. Redis가 현재 연결된 모든 구독자에게 메시지를 전달한다.
4. App B와 App C가 각자의 로컬 캐시를 삭제한다.
5. 끊겨 있던 구독자는 이 메시지를 다시 받을 수 없다.

---

# 메시징 방식 비교

| 항목 | Redis Pub/Sub | Redis Streams | SNS + SQS | Kafka |
|------|---------------|---------------|-----------|-------|
| 메시지 저장 | 없음 | 있음 | 큐에 보관 | 로그에 보관 |
| 재처리 | 불가 | 가능 | 가능 | 가능 |
| 소비 확인 | 없음 | Consumer Group 지원 | SQS 삭제로 확인 | Offset |
| 적합한 용도 | 일시적 알림, 무효화 | 경량 이벤트 처리 | AWS 비동기 통합 | 대규모 이벤트 스트림 |

Pub/Sub은 유실해도 다음 TTL이나 재조회로 복구 가능한 보조 신호에 적합하다.

---

# ElastiCache란?

Amazon ElastiCache는 Valkey, Redis OSS, Memcached 호환 캐시를 제공하는 AWS 관리형 서비스이다.

AWS는 노드 교체, 모니터링 통합, 백업과 Failover 기능을 제공하지만 애플리케이션 데이터 모델과 장애 대응까지 대신 설계하지는 않는다.

---

# Cluster Mode on/off 비교

| 항목 | Cluster Mode 비활성 | Cluster Mode 활성 |
|------|---------------------|-------------------|
| 데이터 분할 | 단일 샤드 | 여러 샤드 |
| 쓰기 확장 | Primary 한 대 한계 | 샤드별 Primary로 분산 |
| 키 연산 | 단일 키 공간이 단순 | 다중 키는 같은 Hash Slot 필요 |
| 확장 목적 | 단순 운영과 복제 | 더 큰 메모리와 처리량 |
| 클라이언트 | 일반 연결 구성 | Cluster topology 인식 필요 |

---

# 고가용성과 Failover

복제본을 다른 AZ에 두고 Multi-AZ 자동 Failover를 활성화하면 Primary 장애 시 복제본이 승격될 수 있다.

복제는 비동기일 수 있으므로 장애 직전의 최근 쓰기가 새 Primary에 없을 가능성을 데이터 특성에 맞게 고려한다.

Failover 과정에서는 기존 연결이 끊기고 DNS 또는 topology가 바뀔 수 있으므로 클라이언트의 재연결 동작과 타임아웃을 시험해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

`RedisMessageListenerContainer`는 연결과 스레드 디스패치를 관리하고 채널 메시지를 리스너에 전달한다.

```java
@Configuration
public class RedisPubSubConfig {

    @Bean
    RedisMessageListenerContainer listenerContainer(
            RedisConnectionFactory connectionFactory,
            MessageListener productChangedListener) {
        RedisMessageListenerContainer container =
                new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(
                productChangedListener,
                new ChannelTopic("product.changed"));
        return container;
    }
}
```

리스너는 메시지를 작게 유지하고 중복 수신에도 같은 결과가 되도록 멱등하게 처리한다.

```java
@Component
public class ProductChangedListener implements MessageListener {
    private final CacheManager cacheManager;

    public ProductChangedListener(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String productId = new String(message.getBody(), StandardCharsets.UTF_8);
        Cache cache = cacheManager.getCache("local-products");
        if (cache != null) {
            cache.evict(productId);
        }
    }
}
```

발행은 `StringRedisTemplate`로 수행한다.

```java
@Service
public class ProductEventPublisher {
    private final StringRedisTemplate redisTemplate;

    public ProductEventPublisher(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void publishChanged(long productId) {
        redisTemplate.convertAndSend("product.changed", Long.toString(productId));
    }
}
```

DB 커밋 전에 발행하면 롤백된 변경을 구독자가 반영할 수 있으므로 `AFTER_COMMIT` 이벤트나 Outbox를 검토한다.

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
      timeout: 1s
```

---

# 실무에서는 어떻게 사용할까

ElastiCache는 애플리케이션과 같은 VPC의 사설 서브넷에 두고 보안 그룹으로 접근 주체를 제한한다.

메모리 사용률, CPU, 연결 수, eviction, 복제 지연, Hit Ratio와 애플리케이션 Redis 오류율을 함께 관찰한다.

---

# 장애 사례

Failover 중 기존 커넥션이 끊겼는데 클라이언트가 재연결하지 못하면 캐시, 세션, 락, 구독이 동시에 실패할 수 있다.

구독 연결이 복구되어도 끊긴 동안의 Pub/Sub 메시지는 재전송되지 않으므로 TTL 또는 주기적 전체 동기화로 수렴시킨다.

메모리 사용량이 한계에 도달하면 eviction이 급증하거나 `noeviction` 정책에서 쓰기 오류가 발생한다.

단일 Hot Key나 큰 값은 특정 샤드의 CPU와 네트워크를 집중시켜 Cluster Mode에서도 병목을 만든다.

---

# 주의할 점

자동 Failover는 무중단과 데이터 무손실을 보장하는 표현이 아니며 연결 끊김과 복제 지연을 전제로 설계해야 한다.

Pub/Sub을 중요한 업무 이벤트의 유일한 전달 경로로 사용하면 소비자가 잠시 중단되었을 때 복구할 수 없다.

캐시, 세션, 분산 락을 한 클러스터에 섞으면 한 워크로드의 장애가 다른 기능으로 전파될 수 있다.

---

# 비용과 성능 고려사항

ElastiCache 비용은 선택한 노드 타입과 노드 수, 복제본, Multi-AZ, 백업 보존, 데이터 전송에 영향을 받는다.

Cluster Mode는 메모리와 처리량을 확장하지만 샤드와 복제본 수만큼 운영 복잡도와 노드 비용이 증가한다.

큰 메시지를 Pub/Sub으로 반복 전송하면 네트워크와 구독자 역직렬화 비용이 커지므로 식별자 중심의 작은 이벤트를 사용한다.

---

# 기억해야 할 내용

- Redis Pub/Sub은 현재 구독자에게 메시지를 전달하지만 저장과 재처리를 보장하지 않는다.
- 중요한 이벤트에는 Streams, SQS, Kafka 같은 내구성 있는 수단을 검토한다.
- ElastiCache는 캐시 엔진의 배포와 일부 운영 작업을 관리형으로 제공한다.
- Cluster Mode 활성화는 데이터를 여러 샤드로 나누어 용량과 처리량을 확장한다.
- Multi-AZ 자동 Failover 중에도 연결 끊김과 최근 쓰기 손실 가능성을 고려한다.
- `RedisMessageListenerContainer`로 Spring Boot 구독자를 구성할 수 있다.
- 메모리, eviction, 연결 수, 복제 지연, 애플리케이션 오류율을 함께 관찰한다.

---

# 다음 Chapter

다음 Chapter는 **Chapter 08. Part 6 Summary**이다.

Cache Aside, Write Through, TTL, Redis Session, Distributed Lock, Pub/Sub과 ElastiCache 운영 원칙을 하나의 설계 관점으로 정리한다.


