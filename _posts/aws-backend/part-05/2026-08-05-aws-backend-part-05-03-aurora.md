---
title: "Chapter 03. Aurora"
permalink: /aws-backend/part-05/03-aurora/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Aurora
## AWS가 만든 관계형 데이터베이스 엔진

> **학습 목표**
>
> - Aurora와 일반 RDS의 차이를 설명할 수 있다.
> - Aurora의 스토리지 구조가 가용성에 주는 영향을 이해한다.
> - Aurora를 선택할 때 고려할 기준을 설명할 수 있다.
> - Writer와 Reader Endpoint의 역할을 구분할 수 있다.

---

# 왜 Aurora가 필요한가

쇼핑몰의 주문과 상품 조회가 늘면 단일 DB 인스턴스의 성능뿐 아니라 장애 복구 시간과 읽기 확장 방식이 중요해진다.

일반적인 DB 구조에서는 인스턴스 장애 시 스토리지 복구나 복제본 승격 과정이 길어질 수 있고 읽기 복제본마다 별도 스토리지를 관리해야 한다.

Spring Boot 서버를 여러 대로 늘려도 모든 요청이 하나의 Writer로 몰리면 데이터베이스가 전체 서비스의 병목이 된다.

Aurora는 컴퓨팅 노드와 클러스터 스토리지를 분리하여 장애 시 다른 인스턴스가 동일한 분산 스토리지를 사용하도록 설계되었다.

이 구조는 모든 시스템에 필요한 것은 아니지만 높은 가용성과 읽기 확장이 중요한 서비스에서 강력한 선택지가 된다.

---

# Aurora란?

Aurora는 AWS가 제공하는 MySQL/PostgreSQL 호환 관계형 데이터베이스 엔진이다.

MySQL 호환 Aurora와 PostgreSQL 호환 Aurora 중 하나를 선택하며 기존 드라이버와 많은 도구를 계속 사용할 수 있다.

호환은 원본 엔진의 모든 기능과 플러그인이 완전히 동일하다는 뜻이 아니므로 마이그레이션 전에 지원 기능을 검증해야 한다.

---

# 구조

Aurora DB 클러스터는 하나의 Writer와 0개 이상의 Aurora Replica가 공유 분산 스토리지를 사용한다.

```
Spring Boot
   ├── 쓰기 ── Cluster Endpoint ── Writer
   │                                │
   └── 읽기 ── Reader Endpoint ─┬── Reader 1
                                └── Reader 2
                                     │
                ┌────────────────────┴───────────────────┐
                │ Aurora Shared Cluster Storage          │
                │ AZ-A 2 copies | AZ-B 2 | AZ-C 2        │
                └─────────────────────────────────────────┘
```

Aurora 스토리지는 일반적으로 세 AZ에 걸쳐 데이터 사본 6개를 유지하는 분산 구조이며 장애 복구와 내구성을 지원한다.

컴퓨팅 인스턴스가 로컬 데이터 전체를 별도로 복제하는 방식이 아니라 공유 스토리지 계층에 접근한다.

---

# 내부 동작 원리

Writer는 `INSERT`, `UPDATE`, `DELETE`와 트랜잭션 커밋을 처리한다.

변경 로그는 분산 스토리지 계층으로 전달되고 스토리지 노드의 쿼럼을 통해 지속성이 확보된다.

Aurora Replica는 같은 클러스터 스토리지를 사용하면서 읽기 요청을 처리하므로 전통적인 엔진 기반 복제보다 복제 부담을 줄인다.

Reader Endpoint는 사용 가능한 Replica들에 새 연결을 분산하지만 이미 열린 연결의 쿼리를 매번 분산하는 로드밸런서는 아니다.

Writer 장애 시 Replica 중 하나가 승격될 수 있고 클러스터 엔드포인트의 DNS 대상이 새 Writer로 변경된다.

---

# 일반 RDS와 Aurora 비교

| 기준 | 일반 RDS MySQL/PostgreSQL | Aurora |
|---|---|---|
| 엔진 | 커뮤니티 엔진 기반 관리형 서비스 | AWS 구현 호환 엔진 |
| 스토리지 | 인스턴스별 관리형 스토리지 | 공유 분산 클러스터 스토리지 |
| 스토리지 복제 | Multi-AZ 구성에 따라 다름 | 여러 AZ에 6-way 복제 |
| 읽기 확장 | Read Replica별 엔드포인트 | Aurora Replica와 Reader Endpoint |
| 장애 전환 | 구성에 따라 대기 인스턴스 승격 | Replica 승격과 공유 스토리지 활용 |
| 엔진 범위 | 여러 관계형 DB 엔진 | MySQL/PostgreSQL 호환 |
| 비용 구조 | 인스턴스, 스토리지 등 | 인스턴스, 스토리지, I/O 방식 등 |

Aurora의 빠른 장애 전환도 애플리케이션 관점에서는 연결 실패 구간이 존재하므로 무중단을 의미하지 않는다.

---

# 엔드포인트 종류

| 엔드포인트 | 연결 대상 | 사용 목적 |
|---|---|---|
| Cluster Endpoint | 현재 Writer | 쓰기와 강한 정합성 읽기 |
| Reader Endpoint | 사용 가능한 Replica | 읽기 부하 분산 |
| Instance Endpoint | 특정 인스턴스 | 진단 또는 특수 라우팅 |
| Custom Endpoint | 지정한 인스턴스 그룹 | 워크로드별 Reader 분리 |

Reader Endpoint에 쓰기 SQL을 보내면 실패하며 Reader가 하나도 없으면 기대한 읽기 확장을 얻을 수 없다.

---

# AWS 콘솔/CLI에서는

클러스터 생성 시 엔진 호환 버전, 인스턴스 클래스, Replica 수와 AZ 배치, 백업, 암호화, 삭제 방지를 결정한다.

```bash
aws rds describe-db-clusters \
  --db-cluster-identifier shop-aurora \
  --query 'DBClusters[0].{Status:Status,Writer:Endpoint,Reader:ReaderEndpoint,MultiAZ:MultiAZ}'
```

클러스터 구성원별 Writer 여부와 승격 우선순위도 함께 확인한다.

```bash
aws rds describe-db-instances \
  --filters Name=db-cluster-id,Values=shop-aurora \
  --query 'DBInstances[].{Id:DBInstanceIdentifier,AZ:AvailabilityZone,Status:DBInstanceStatus}'
```

---

# Spring Boot에서는 어떻게 쓰는가

쓰기 중심의 단순한 서비스는 Cluster Endpoint 하나로 시작하고 읽기 분리가 필요할 때 Reader 데이터소스를 추가한다.

```yaml
app:
  datasource:
    writer:
      url: jdbc:postgresql://${AURORA_WRITER}:5432/shop
      username: ${DB_USERNAME}
      password: ${DB_PASSWORD}
    reader:
      url: jdbc:postgresql://${AURORA_READER}:5432/shop
      username: ${DB_USERNAME}
      password: ${DB_PASSWORD}
spring:
  jpa:
    open-in-view: false
```

Reader Endpoint는 연결 생성 시 분산되므로 HikariCP 연결 수가 너무 적거나 연결 수명이 지나치게 길면 Reader 간 부하가 고르게 이동하지 않을 수 있다.

```java
@Service
public class CatalogService {

    private final ProductRepository productRepository;

    public CatalogService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional(readOnly = true)
    public List<Product> findProducts() {
        return productRepository.findAll();
    }
}
```

실제 Writer/Reader 라우팅은 `AbstractRoutingDataSource` 같은 명시적 구성이 필요하며 `readOnly = true`만 선언한다고 자동으로 Reader Endpoint에 연결되지는 않는다.

---

# 실무에서는 어떻게 사용할까

주문 생성과 결제 상태 변경은 Writer Endpoint를 사용하고 상품 목록, 통계처럼 지연을 허용하는 조회는 Reader Endpoint로 분리한다.

쓰기 직후 자신의 주문을 확인하는 흐름은 Writer에서 읽거나 애플리케이션 수준의 일관성 전략을 사용한다.

Reader를 서로 다른 AZ에 배치하고 장애 전환 우선순위를 정한 뒤 실제 장애 주입으로 복구 시간을 측정한다.

Aurora Serverless를 검토할 때도 부하 패턴, 확장 지연, 최소·최대 용량, 연결 특성을 별도로 검증해야 한다.

---

# 장애 사례와 주의할 점

Reader Endpoint가 SQL 단위로 부하를 분산한다고 오해하면 하나의 장기 연결에서 실행되는 대량 조회가 특정 Reader에 집중될 수 있다.

Failover 후 커넥션 풀이 이전 Writer 연결을 보관하면 새 Writer가 준비되어도 요청이 계속 실패할 수 있다.

MySQL 또는 PostgreSQL 호환이라는 이유만으로 확장 기능과 버전 차이를 확인하지 않으면 마이그레이션 중 호환성 문제가 발생한다.

Replica가 없는 클러스터에서는 Writer 장애 시 새 컴퓨팅 인스턴스 준비가 필요해 Replica가 준비된 구성과 복구 특성이 다를 수 있다.

---

# 비용과 성능 고려사항

Aurora 비용은 DB 인스턴스 실행 시간, 스토리지 사용량, I/O 과금 방식, 백업 보관량, 데이터 전송, Replica 수에 영향을 받는다.

일반 RDS보다 높은 가용성과 운영 기능이 필요한지 확인하고 실제 쿼리 기반 부하 테스트로 가격 대비 효과를 판단해야 한다.

Reader 추가는 읽기 용량을 늘리지만 Writer의 쓰기 병목이나 비효율적인 쿼리를 자동으로 해결하지 않는다.

---

# 기억해야 할 내용

- Aurora는 MySQL 또는 PostgreSQL과 호환되는 AWS의 관계형 DB 엔진이다.
- 컴퓨팅과 여러 AZ의 분산 클러스터 스토리지가 분리되어 있다.
- 스토리지는 세 AZ에 걸쳐 6개 사본을 유지하는 구조이다.
- Cluster Endpoint는 Writer, Reader Endpoint는 Replica 연결에 사용한다.
- Reader Endpoint는 연결을 분산하며 쿼리를 매번 분산하지 않는다.
- 호환성, 가용성, 성능, 비용을 실제 워크로드로 검증해야 한다.
- 빠른 Failover도 애플리케이션 재연결 설계가 필요하다.

---

# 다음 Chapter

다음 Chapter에서는 여러 AZ에 데이터베이스를 배치해 가용성을 높이는 [Multi-AZ](/aws-backend/part-05/04-multi-az/)를 학습한다.

일반 RDS Multi-AZ DB 인스턴스, RDS Multi-AZ DB 클러스터, Aurora의 차이를 구분한다.
