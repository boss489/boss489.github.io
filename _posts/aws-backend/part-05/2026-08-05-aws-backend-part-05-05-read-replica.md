---
title: "Chapter 05. Read Replica"
permalink: /aws-backend/part-05/05-read-replica/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Read Replica
## 읽기 부하를 분산하는 복제본

> **학습 목표**
>
> - Read Replica의 목적을 설명할 수 있다.
> - 복제 지연의 의미를 이해한다.
> - 쓰기와 읽기 트래픽 분리 기준을 설명할 수 있다.
> - Spring Boot에서 Writer와 Reader 데이터소스를 라우팅할 수 있다.

---

# 왜 Read Replica가 필요한가

쇼핑몰에서 상품 목록, 주문 내역, 관리자 통계 조회가 늘면 쓰기보다 읽기 쿼리가 DB 자원을 더 많이 사용할 수 있다.

모든 조회가 Primary로 향하면 대량 통계 쿼리가 주문 생성과 재고 차감 트랜잭션의 응답 시간까지 늦춘다.

Spring Boot API 서버만 Auto Scaling해도 데이터베이스의 읽기 처리 용량은 자동으로 늘어나지 않는다.

Read Replica는 읽기 가능한 복제본을 추가하여 Primary의 조회 부하를 분산하는 수평 확장 수단이다.

---

# Read Replica란?

Read Replica는 Primary 데이터베이스의 데이터를 복제해 읽기 요청을 처리하는 DB 인스턴스다.

주 목적은 읽기 확장이다.

일반적으로 Primary의 변경 로그를 비동기적으로 적용하므로 Reader의 데이터가 잠시 과거 상태일 수 있다.

Read Replica는 고가용성을 위한 Standby와 달리 고유 엔드포인트를 가지며 애플리케이션이 읽기 요청을 명시적으로 보내야 한다.

---

# 구조와 내부 동작

```
                 ┌── 쓰기/즉시 읽기 ── Writer Endpoint
Spring Boot API ─┤                         │
                 └── 지연 허용 읽기 ── Reader Endpoint
                                           │
                 ┌─────────────────────────┼────────────┐
                 │                         │            │
              Primary ── 비동기 복제 ── Replica 1   Replica 2
```

Primary가 트랜잭션을 커밋하면 변경 내용이 복제 스트림에 기록된다.

Replica는 변경 내용을 전달받아 자신의 데이터에 순서대로 적용한다.

네트워크, 대량 쓰기, 긴 쿼리, Replica 자원 부족이 있으면 적용 속도가 쓰기 속도를 따라가지 못해 지연이 커진다.

RDS의 일반 Read Replica는 보통 각 Replica 엔드포인트를 애플리케이션이 선택하며 Aurora는 Reader Endpoint로 새 연결을 Replica들에 분산할 수 있다.

---

# 복제 지연과 정합성

Read Replica는 Primary와 완전히 동시에 반영되지 않을 수 있다.

이 지연을 Replication Lag라고 한다.

방금 쓴 데이터를 즉시 읽어야 하는 요청은 Primary를 읽어야 한다.

예를 들어 주문 생성 직후 Replica에서 주문 상세를 조회하면 아직 주문이 보이지 않아 잘못된 `404`를 반환할 수 있다.

| 조회 유형 | 권장 대상 | 이유 |
|---|---|---|
| 주문 생성 직후 상세 | Writer | Read-after-write 정합성 필요 |
| 결제 상태 확인 | Writer 또는 정합성 전략 | 오래된 상태의 영향이 큼 |
| 상품 목록 | Reader | 짧은 지연 허용 가능 |
| 관리자 일간 통계 | Reader | 최신성보다 부하 분리가 중요 |
| 재고 차감 판단 | Writer | 동시성 제어와 최신 값 필요 |

`@Transactional(readOnly = true)`는 쓰기 의도를 표현하고 최적화 힌트가 될 수 있지만 그 자체로 복제 지연을 제거하지 않는다.

---

# Multi-AZ와 비교

| 기준 | Multi-AZ DB 인스턴스 | Read Replica |
|---|---|---|
| 목적 | 고가용성 | 읽기 확장 |
| 복제 방식 | 동기 | 일반적으로 비동기 |
| 읽기 가능 | Standby 불가 | 가능 |
| 애플리케이션 연결 | 하나의 Primary 엔드포인트 | Reader 엔드포인트 별도 |
| 지연 고려 | 커밋 지연 | Replication Lag |
| 장애 역할 | 자동 Failover | 자동 대체 여부는 구성별 확인 |

Read Replica를 추가했다고 Primary 장애가 자동으로 완전히 해결된다고 가정하면 안 된다.

---

# AWS 콘솔/CLI에서는

Replica 생성 전에 엔진 버전, 백업 설정, 리전과 네트워크, 인스턴스 크기, 암호화 조건을 확인한다.

```bash
aws rds describe-db-instances \
  --query 'DBInstances[].{Id:DBInstanceIdentifier,Source:ReadReplicaSourceDBInstanceIdentifier,Status:DBInstanceStatus,Endpoint:Endpoint.Address}'
```

CloudWatch에서 엔진에 맞는 복제 지연 지표와 CPU, I/O, 여유 메모리, 연결 수를 함께 관찰한다.

Replica 지연이 커질 때 단순히 Replica 수만 늘리지 말고 느린 쿼리와 인스턴스 용량, 쓰기 급증을 확인한다.

---

# Spring Boot에서는 어떻게 쓰는가

Writer와 Reader에 별도 Hikari 데이터소스를 만들고 현재 트랜잭션의 읽기 전용 여부에 따라 라우팅할 수 있다.

```java
public final class TransactionRoutingDataSource
        extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager
                .isCurrentTransactionReadOnly()
                ? "reader"
                : "writer";
    }
}
```

라우팅 데이터소스를 JPA의 기본 데이터소스로 등록할 때 실제 연결 획득을 트랜잭션 판정 이후로 늦추기 위해 `LazyConnectionDataSourceProxy`를 함께 검토한다.

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional(readOnly = true)
    public List<Product> findCatalog() {
        return productRepository.findAll();
    }

    @Transactional
    public Product changePrice(long productId, BigDecimal price) {
        Product product = productRepository.getReferenceById(productId);
        product.changePrice(price);
        return product;
    }
}
```

```yaml
app:
  datasource:
    writer:
      url: jdbc:postgresql://${DB_WRITER}:5432/shop
    reader:
      url: jdbc:postgresql://${DB_READER}:5432/shop
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

클래스 수준의 `readOnly = true`와 쓰기 메서드의 재정의, 트랜잭션 전파, 비동기 실행 시 컨텍스트를 테스트해야 한다.

---

# 실무에서는 어떻게 사용할까

- 상품 목록 조회
- 리포트 조회
- 관리자 통계
- 검색 보조 데이터 조회

이런 조회는 짧은 지연을 허용할 수 있는지 업무 담당자와 합의한 뒤 Reader로 보낸다.

주문·결제처럼 쓰기 직후 읽기가 필요한 흐름은 일정 시간 Writer를 읽거나 버전 토큰을 사용해 최신성 조건을 확인한다.

대량 배치 조회는 사용자 API와 별도 Replica로 분리하여 서로의 연결과 I/O를 잠식하지 않게 할 수 있다.

Aurora에서는 Reader Endpoint 외에 Custom Endpoint로 특정 인스턴스 그룹에 분석 트래픽을 격리할 수 있다.

---

# 장애 사례와 주의할 점

주문 저장 후 바로 Reader를 조회하여 “주문 없음”으로 판단하면 사용자가 같은 주문을 다시 제출할 수 있다.

Replica 지연 경보 없이 통계와 사용자 조회를 모두 보내면 오래된 데이터가 장시간 노출되어도 알아차리기 어렵다.

Writer 트랜잭션 내부에서 호출한 읽기 메서드가 Reader로 전환되면 아직 커밋되지 않은 변경을 볼 수 없으므로 라우팅 전파 규칙을 명확히 해야 한다.

Reader Endpoint가 SQL마다 분산한다고 오해하면 장기 커넥션 때문에 특정 Aurora Replica에 부하가 집중될 수 있다.

---

# 비용과 성능 고려사항

Replica마다 DB 인스턴스 실행 시간, 스토리지 또는 I/O, 백업, 리전 간 데이터 전송 비용이 추가될 수 있다.

읽기 비율과 쿼리 비용을 측정한 뒤 필요한 수와 크기를 정하고 유휴 Replica를 관성적으로 유지하지 않는다.

캐시가 더 적합한 반복 조회도 있으므로 다음 Part의 Redis와 함께 데이터 최신성, 비용, 부하 감소 효과를 비교해야 한다.

Replica는 느린 쿼리를 복제할 뿐이므로 인덱스와 쿼리 개선이 우선이다.

---

# 기억해야 할 내용

- Read Replica의 주 목적은 읽기 성능 확장이다.
- 일반적으로 비동기 복제이므로 Replication Lag가 발생할 수 있다.
- 쓰기 직후 읽기와 정합성이 중요한 판단은 Writer를 사용한다.
- Multi-AZ Standby와 Read Replica의 목적과 읽기 가능 여부는 다르다.
- `readOnly = true` 기반 라우팅에는 별도 데이터소스 구성이 필요하다.
- Aurora Reader Endpoint는 새 연결을 Replica에 분산한다.
- Replica 추가 전에 쿼리와 인덱스를 먼저 점검한다.

---

# 다음 Chapter

다음 Chapter에서는 데이터 손실 이후 원하는 시점으로 돌아가기 위한 [Backup and Restore](/aws-backend/part-05/06-backup-restore/)를 학습한다.

자동 백업과 스냅샷의 차이, RPO와 RTO, 복구 리허설의 필요성을 살펴본다.
