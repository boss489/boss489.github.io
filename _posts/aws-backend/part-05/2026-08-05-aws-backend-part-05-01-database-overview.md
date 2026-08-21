---
title: "Chapter 01. Database Overview"
permalink: /aws-backend/part-05/01-database-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Database Overview
## AWS에서 데이터베이스를 운영한다는 것

> **학습 목표**
>
> - 상태를 가진 데이터베이스가 컴퓨팅 리소스와 다른 이유를 설명할 수 있다.
> - RDS와 Aurora의 역할과 선택 기준을 이해한다.
> - Multi-AZ와 Read Replica의 목적을 구분할 수 있다.
> - Backup, Restore, Failover가 해결하는 문제를 설명할 수 있다.

---

# 왜 데이터베이스 운영을 따로 배워야 하는가

쇼핑몰의 Spring Boot API 서버가 한 대 고장 나면 Auto Scaling이 새 서버를 시작하고 트래픽을 다시 분산할 수 있다.

그러나 주문과 결제 데이터가 저장된 데이터베이스를 잃으면 새 인스턴스를 실행하는 것만으로는 서비스를 복구할 수 없다.

API 서버는 요청을 처리하는 **무상태 컴퓨팅 리소스**로 설계할 수 있지만 데이터베이스는 이전 요청의 결과를 계속 보존하는 **상태 저장 리소스**이다.

예를 들어 운영 데이터베이스 한 대에 장애가 발생하면 상품 조회뿐 아니라 주문 생성, 재고 차감, 결제 결과 저장까지 동시에 멈출 수 있다.

트래픽 증가로 조회 쿼리가 몰리면 주문 쓰기 쿼리까지 느려질 수 있고, 운영자의 잘못된 `DELETE`는 정상 인스턴스가 실행 중이어도 데이터를 훼손한다.

따라서 데이터베이스 운영은 단순히 DB 프로세스를 실행하는 일이 아니라 **가용성, 확장성, 복구 가능성, 정합성**을 함께 설계하는 일이다.

---

# AWS의 관계형 데이터베이스란?

관계형 데이터베이스는 테이블, 관계, 제약 조건, 트랜잭션을 사용해 주문처럼 정합성이 중요한 데이터를 저장한다.

AWS에서는 EC2에 MySQL이나 PostgreSQL을 직접 설치할 수도 있지만, 일반적으로 관리형 서비스인 RDS 또는 Aurora를 사용한다.

RDS는 여러 상용·오픈 소스 관계형 데이터베이스 엔진의 프로비저닝, 백업, 패치, 모니터링 작업을 관리형으로 제공한다.

Aurora는 MySQL 또는 PostgreSQL과 호환되면서 컴퓨팅과 분산 스토리지 계층을 분리한 AWS의 관계형 데이터베이스 엔진이다.

![Database operations](/assets/aws-backend/database-operations.png)

---

# 전체 구조

Part 4에서 ALB가 Spring Boot 서버로 요청을 전달하는 흐름을 배웠다면, 이번 Part에서는 서버가 요청 결과를 안전하게 저장하는 구간을 다룬다.

```
사용자
  │
 ALB
  │
Spring Boot API
  │ JDBC 연결
  ▼
DB Endpoint
  │
  ├── Writer / Primary ── 동기 복제 ── Standby
  │          │
  │          └── 비동기 복제 ── Read Replica
  │
  └── 자동 백업 / 수동 스냅샷
```

Spring Boot는 고정된 DB 인스턴스 IP가 아니라 AWS가 제공하는 **엔드포인트 DNS 이름**으로 접속한다.

Primary 또는 Writer는 쓰기와 강한 정합성이 필요한 읽기를 처리한다.

Standby는 장애 전환을 위한 대기 인스턴스이며, 일반적인 RDS Multi-AZ DB 인스턴스 구성에서는 애플리케이션의 읽기 요청을 처리하지 않는다.

Read Replica는 비동기 복제된 데이터를 읽어 조회 부하를 분산하지만 복제 지연이 발생할 수 있다.

백업은 논리적 실수나 데이터 손상 이후 과거 시점으로 돌아갈 수 있는 별도의 복구 수단이다.

---

# 운영 기능은 어떤 문제를 해결하는가

| 운영 문제 | AWS 기능 | 핵심 목적 |
|---|---|---|
| 서버 또는 AZ 장애 | Multi-AZ | 가용성 확보 |
| 읽기 트래픽 증가 | Read Replica | 읽기 성능 확장 |
| 삭제·손상·재해 | 자동 백업, 스냅샷 | 데이터 복구 |
| Primary 장애 | Failover | 대기 노드로 역할 전환 |
| DB 운영 부담 | RDS | 반복 운영 작업 자동화 |
| 높은 처리량과 빠른 복구 | Aurora | 분산 스토리지 기반 운영 |

Multi-AZ와 Read Replica는 복제를 사용한다는 점은 비슷하지만 목적과 정합성 특성이 다르다.

백업과 Standby도 서로 대체할 수 없는데, 잘못된 데이터 변경은 Standby에도 복제되므로 과거 백업이 필요하다.

---

# RDS와 Aurora 선택 지도

| 기준 | 일반 RDS 엔진 | Aurora |
|---|---|---|
| 엔진 | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server 등 | MySQL, PostgreSQL 호환 |
| 스토리지 | DB 인스턴스에 연결된 관리형 스토리지 | 컴퓨팅과 분리된 분산 클러스터 스토리지 |
| 고가용성 | Multi-AZ 구성 선택 | 여러 AZ에 걸친 스토리지 복제가 기본 구조 |
| 읽기 확장 | 엔진별 Read Replica | Aurora Replica와 Reader Endpoint |
| 운영 복잡도 | 비교적 단순한 구성 가능 | 클러스터와 인스턴스 역할 이해 필요 |
| 선택 기준 | 엔진 호환성, 작은 규모, 기존 운영 경험 | 높은 가용성, 읽기 확장, 빠른 장애 복구 요구 |

Aurora가 항상 정답은 아니며 워크로드, 엔진 기능 호환성, 팀의 운영 역량, 비용 구조를 함께 평가해야 한다.

---

# AWS 콘솔/CLI에서는

데이터베이스를 만들기 전에 VPC, 최소 두 AZ의 DB Subnet Group, 보안 그룹, 파라미터 그룹, 백업 보관 기간을 결정한다.

AWS CLI에서는 다음과 같이 현재 리전의 RDS 인스턴스 상태와 엔드포인트를 확인할 수 있다.

```bash
aws rds describe-db-instances \
  --query 'DBInstances[].{Id:DBInstanceIdentifier,Engine:Engine,Status:DBInstanceStatus,MultiAZ:MultiAZ,Endpoint:Endpoint.Address}'
```

Aurora를 포함한 DB 클러스터는 별도의 명령으로 Writer 여부와 상태를 확인한다.

```bash
aws rds describe-db-clusters \
  --query 'DBClusters[].{Id:DBClusterIdentifier,Engine:Engine,Status:Status,Endpoint:Endpoint,Reader:ReaderEndpoint}'
```

운영 환경에서는 콘솔 화면만 보지 말고 구성 정보를 코드 또는 정기 점검 항목으로 관리해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

Spring Boot 3.x 애플리케이션은 JDBC 드라이버와 HikariCP 커넥션 풀을 통해 RDS 또는 Aurora 엔드포인트에 연결한다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/shop
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000
      validation-timeout: 1000
  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
```

접속 정보는 소스 코드에 넣지 않고 Secrets Manager 같은 비밀 저장소 또는 배포 환경 변수로 주입한다.

커넥션 풀 크기는 애플리케이션 인스턴스 수까지 곱해서 데이터베이스의 최대 연결 수를 넘지 않도록 계산한다.

`@Transactional`은 서비스 계층에서 트랜잭션 경계를 명확히 하고, 긴 외부 API 호출을 트랜잭션 안에 넣지 않도록 설계한다.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public Order createOrder(Order order) {
        return orderRepository.save(order);
    }
}
```

---

# 실무에서는 어떻게 사용할까

소규모 쇼핑몰은 RDS PostgreSQL 단일 Writer와 Multi-AZ로 시작하고, 백업 보관 및 복구 절차를 먼저 확립할 수 있다.

상품 목록이나 통계 조회가 쓰기 처리량을 압박하면 Read Replica를 추가하고 읽기 전용 트랜잭션만 분리한다.

높은 읽기 처리량과 빠른 장애 전환이 중요해지면 Aurora 클러스터를 검토하되 실제 쿼리와 장애 테스트로 효과를 확인한다.

운영 지표는 CPU뿐 아니라 DB 연결 수, 여유 메모리, 스토리지 공간, IOPS, 지연 시간, 잠금, 복제 지연을 함께 관찰한다.

---

# 장애 사례와 주의할 점

Multi-AZ를 켰다는 이유로 백업을 생략하면 운영자가 잘못 삭제한 데이터가 Standby에도 그대로 복제되어 복구할 수 없다.

Read Replica에서 주문 직후 주문 상세를 조회하면 복제 지연 때문에 방금 생성한 주문이 보이지 않을 수 있다.

애플리케이션 인스턴스를 자동 확장하면서 각 인스턴스의 커넥션 풀을 크게 잡으면 DB 연결 한도를 소진해 정상 요청도 실패한다.

DB를 Public Subnet에 두거나 보안 그룹에서 인터넷 전체를 허용하면 자격 증명 공격과 데이터 노출 위험이 커진다.

---

# 비용과 성능 고려사항

주요 과금 요소는 DB 인스턴스 실행 시간, 프로비저닝 또는 사용한 스토리지, I/O, 백업 보관량, 데이터 전송, 추가 복제본이다.

Multi-AZ와 Read Replica는 인스턴스와 스토리지 자원을 추가하므로 비용이 늘지만 각각 가용성과 읽기 처리량이라는 다른 가치를 제공한다.

과도한 인스턴스 확장 전에 느린 쿼리, 누락된 인덱스, N+1 조회, 불필요하게 긴 트랜잭션을 먼저 개선해야 한다.

---

# 기억해야 할 내용

- 데이터베이스는 교체하기 어려운 상태 저장 리소스이다.
- RDS는 관계형 데이터베이스의 반복 운영 작업을 관리형으로 제공한다.
- Aurora는 MySQL/PostgreSQL 호환 엔진과 분산 스토리지 구조를 제공한다.
- Multi-AZ는 고가용성, Read Replica는 읽기 확장이 목적이다.
- Standby는 백업을 대체하지 않으며 백업은 복구 테스트까지 해야 한다.
- Spring Boot의 전체 커넥션 수는 애플리케이션 수와 함께 계산해야 한다.
- 비용과 성능은 인스턴스뿐 아니라 스토리지, I/O, 연결, 쿼리까지 함께 봐야 한다.

---

# 다음 Chapter

다음 Chapter에서는 관계형 데이터베이스를 관리형으로 제공하는 [RDS](/aws-backend/part-05/02-rds/)를 학습한다.

EC2에 DB를 직접 설치하는 방식과 비교해 AWS가 담당하는 영역과 사용자가 여전히 책임져야 하는 영역을 구분한다.
