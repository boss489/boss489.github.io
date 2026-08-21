---
title: "Chapter 04. Multi AZ"
permalink: /aws-backend/part-05/04-multi-az/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Multi AZ
## 데이터베이스 고가용성 구성

> **학습 목표**
>
> - Multi-AZ의 목적을 설명할 수 있다.
> - Standby와 Failover의 관계를 이해한다.
> - Multi-AZ가 읽기 확장 목적이 아님을 설명할 수 있다.
> - RDS Multi-AZ 구성과 Aurora의 차이를 구분할 수 있다.

---

# 왜 Multi-AZ가 필요한가

쇼핑몰 주문 DB가 하나의 AZ에만 있으면 DB 인스턴스나 AZ의 전원·네트워크 문제로 모든 주문 API가 중단될 수 있다.

Spring Boot 서버를 여러 AZ에 배치해도 데이터베이스가 단일 장애 지점이면 전체 서비스는 고가용성이 아니다.

담당자가 장애 후 새 인스턴스를 만들고 백업을 복원하는 방식은 데이터 손실과 긴 중단 시간을 피하기 어렵다.

Multi-AZ는 정상 시점부터 다른 AZ에 장애 전환 대상을 준비하고 변경 사항을 동기적으로 복제하여 이 문제를 줄인다.

---

# Multi-AZ란?

Multi-AZ는 데이터베이스를 여러 Availability Zone에 배치해 장애에 대비하는 구성이다.

Primary에 장애가 발생하면 Standby로 Failover한다.

![Database operations](/assets/aws-backend/database-operations.png)

일반적인 RDS Multi-AZ DB 인스턴스 배포에서 Standby는 고가용성을 위한 대기 인스턴스이며 읽기 트래픽에 사용하지 않는다.

Multi-AZ는 제품 이름 하나가 아니라 RDS 엔진과 배포 유형에 따라 구조가 다른 고가용성 선택지이다.

---

# 구조와 내부 동작

```
Availability Zone A              Availability Zone C
┌──────────────────┐            ┌──────────────────┐
│ Spring Boot API  │            │ Spring Boot API  │
└────────┬─────────┘            └────────┬─────────┘
         └───────────┬───────────────────┘
                     │ 하나의 DB Endpoint
              ┌──────▼──────┐
              │ Primary DB  │
              │ Read/Write  │
              └──────┬──────┘
                     │ 동기 복제
              ┌──────▼──────┐
              │ Standby DB  │
              │ 읽기 불가   │
              └─────────────┘
```

애플리케이션은 Primary와 Standby의 주소를 직접 선택하지 않고 하나의 RDS 엔드포인트에 연결한다.

트랜잭션 변경은 Primary에서 처리되고 고가용성 구조의 Standby로 동기 복제된다.

AWS가 장애를 감지하면 Standby를 새 Primary로 승격하고 엔드포인트 DNS가 새 대상을 가리키도록 변경한다.

애플리케이션은 DNS를 다시 조회하고 새 DB에 연결해야 하며 기존 TCP 연결은 그대로 이동하지 않는다.

---

# 배포 유형을 구분해야 한다

| 구성 | 인스턴스 구조 | Standby 읽기 | 핵심 특징 |
|---|---|---|---|
| RDS Multi-AZ DB 인스턴스 | Primary + 다른 AZ Standby | 불가 | 전통적인 고가용성 구성 |
| RDS Multi-AZ DB 클러스터 | Writer + 읽기 가능한 두 인스턴스 | 가능 | 일부 엔진에서 제공되는 클러스터형 배포 |
| Aurora DB 클러스터 | Writer/Reader + 공유 분산 스토리지 | Reader 가능 | 스토리지 계층이 여러 AZ에 복제 |
| Read Replica | Primary + 비동기 Replica | 가능 | 읽기 확장이 주 목적 |

“Multi-AZ의 Standby는 읽을 수 없다”는 설명은 RDS Multi-AZ DB 인스턴스 배포에는 맞지만 모든 클러스터형 구성에 일괄 적용하면 안 된다.

---

# Multi-AZ와 Read Replica 비교

| 기준 | Multi-AZ DB 인스턴스 | Read Replica |
|---|---|---|
| 주 목적 | 고가용성 | 읽기 확장 |
| 복제 | 동기식 | 일반적으로 비동기식 |
| 읽기 요청 | Standby 사용 불가 | Replica 사용 가능 |
| 장애 처리 | 자동 Failover 지원 | 승격 절차와 운영 설계 필요 |
| 엔드포인트 | 장애 전환 시 동일 엔드포인트 사용 | Replica별 엔드포인트 |
| 복제 지연 | 커밋 경로에 반영 | 지연 가능 |

두 기능을 함께 사용하여 Primary 고가용성과 조회 확장을 동시에 구성할 수도 있다.

---

# Failover가 발생하는 상황

Failover는 다음 상황에서 발생할 수 있다.

- DB 인스턴스 장애
- AZ 장애
- 계획된 유지보수
- 네트워크 장애
- 사용자가 재부팅 시 Failover를 선택한 경우

모든 성능 저하나 SQL 오류가 Failover를 유발하는 것은 아니며 정확한 조건은 엔진과 배포 유형에 따라 다르다.

---

# AWS 콘솔/CLI에서는

DB 생성 또는 수정 화면에서 Multi-AZ 배포를 선택하고 DB Subnet Group에 여러 AZ의 서브넷이 포함되었는지 확인한다.

```bash
aws rds describe-db-instances \
  --db-instance-identifier shop-prod-db \
  --query 'DBInstances[0].{Status:DBInstanceStatus,MultiAZ:MultiAZ,AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone,Endpoint:Endpoint.Address}'
```

운영 전에는 유지보수 시간에 재부팅 Failover를 실행하여 애플리케이션 오류율과 복구 시간을 측정할 수 있다.

```bash
aws rds reboot-db-instance \
  --db-instance-identifier shop-prod-db \
  --force-failover
```

이 명령은 실제 연결 장애를 일으키므로 운영 승인과 영향 통제 없이 실행하면 안 된다.

---

# Spring Boot에서는 어떻게 쓰는가

애플리케이션은 Primary 인스턴스의 고정 IP가 아니라 RDS가 제공하는 엔드포인트를 사용한다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${RDS_ENDPOINT}:5432/shop
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      connection-timeout: 3000
      validation-timeout: 1000
      keepalive-time: 120000
  jpa:
    open-in-view: false
```

Failover 중 실행되던 트랜잭션은 결과를 알 수 없거나 롤백될 수 있으므로 단순히 모든 요청을 무조건 재시도하면 안 된다.

주문 생성 재시도에는 멱등성 키나 유일 제약 조건을 사용하여 중복 주문을 방지한다.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public Order createOrder(String idempotencyKey, Order order) {
        orderRepository.findByIdempotencyKey(idempotencyKey)
                .ifPresent(existing -> {
                    throw new DuplicateOrderRequestException(idempotencyKey);
                });
        return orderRepository.save(order.withIdempotencyKey(idempotencyKey));
    }
}
```

---

# 실무에서는 어떻게 사용할까

주문, 결제, 회원처럼 중단 영향이 큰 운영 DB는 Multi-AZ를 기본 후보로 검토한다.

개발 환경까지 같은 수준으로 구성할지는 비용과 테스트 목적을 기준으로 결정하되 운영과 동일한 Failover 검증 환경은 확보한다.

CloudWatch 이벤트와 애플리케이션의 DB 연결 오류, 트랜잭션 실패율, 복구 시간을 같은 타임라인으로 관찰한다.

장애 대응 문서에는 Failover 완료 확인, 쓰기 검증, 지연 작업 재처리, 사용자 공지 기준을 포함한다.

---

# 장애 사례와 주의할 점

Multi-AZ를 활성화했지만 애플리케이션이 과거 IP를 캐시하면 새 Primary로 연결하지 못한다.

커넥션 풀이 죽은 연결을 계속 빌려주면 DNS 전환이 끝난 뒤에도 요청 실패가 이어질 수 있다.

동기 복제는 가용성을 높이지만 장거리 재해 복구나 운영자의 논리적 삭제를 막는 백업은 아니다.

Failover 테스트를 하지 않으면 JDBC 드라이버, DNS 캐시, 재시도 정책의 실제 복구 시간을 알 수 없다.

---

# 비용과 성능 고려사항

Multi-AZ는 추가 DB 인스턴스와 스토리지·I/O 구조 때문에 Single-AZ보다 비용이 증가한다.

동기 복제는 쓰기 커밋 지연에 영향을 줄 수 있으므로 운영과 유사한 쓰기 부하로 성능을 검증해야 한다.

비용 비교에는 인프라 요금뿐 아니라 장애 중 주문 손실, 복구 인력, SLA 위반 위험도 포함해야 한다.

---

# 기억해야 할 내용

- Multi-AZ의 주 목적은 읽기 확장이 아니라 고가용성이다.
- 일반 RDS Multi-AZ DB 인스턴스의 Standby는 읽기에 사용할 수 없다.
- RDS Multi-AZ DB 클러스터와 Aurora는 구조와 읽기 가능 여부가 다르다.
- 애플리케이션은 고정 IP가 아니라 RDS 엔드포인트를 사용해야 한다.
- Failover 중 기존 연결과 진행 중 트랜잭션은 실패할 수 있다.
- Multi-AZ는 백업과 Read Replica를 대체하지 않는다.
- 운영 전에 애플리케이션을 포함한 Failover 테스트가 필요하다.

---

# 다음 Chapter

다음 Chapter에서는 비동기 복제를 이용해 읽기 부하를 분산하는 [Read Replica](/aws-backend/part-05/05-read-replica/)를 학습한다.

Multi-AZ와 목적이 어떻게 다르며 복제 지연을 애플리케이션에서 어떻게 다뤄야 하는지 살펴본다.
