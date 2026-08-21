---
title: "Chapter 07. Failover"
permalink: /aws-backend/part-05/07-failover/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Failover
## 장애 시 다른 DB로 전환하기

> **학습 목표**
>
> - Failover의 의미를 설명할 수 있다.
> - Failover 중 애플리케이션에 발생할 수 있는 영향을 이해한다.
> - 연결 풀과 재시도 설정의 중요성을 설명할 수 있다.
> - DNS와 트랜잭션을 고려한 장애 대응 절차를 설계할 수 있다.

---

# 왜 Failover를 애플리케이션까지 고려해야 하는가

쇼핑몰의 Primary DB가 멈추면 AWS가 Standby를 승격하더라도 실행 중인 주문 요청의 연결은 새 인스턴스로 이동하지 않는다.

Spring Boot의 HikariCP에는 이전 Primary로 연결된 TCP 세션이 남고 진행 중 트랜잭션은 실패하거나 결과를 확정하기 어려울 수 있다.

애플리케이션이 DB IP를 고정하거나 DNS를 오래 캐시하면 승격이 끝난 뒤에도 새 Primary를 찾지 못한다.

따라서 Failover는 DB 내부 이벤트가 아니라 DNS, JDBC, 커넥션 풀, 재시도, 사용자 응답을 포함한 전체 시스템 사건이다.

---

# Failover란?

Failover는 Primary DB에 장애가 발생했을 때 Standby 또는 다른 노드로 역할을 넘기는 과정이다.

RDS Multi-AZ DB 인스턴스는 Standby를 Primary로 전환하고 기존 엔드포인트가 새 대상을 가리키도록 변경한다.

Aurora는 Aurora Replica를 Writer로 승격할 수 있으며 클러스터 엔드포인트가 새 Writer를 가리키도록 변경된다.

Failover는 중단 시간을 줄이지만 진행 중 연결과 트랜잭션을 보존하는 무중단 기능은 아니다.

---

# 내부 동작 원리

```
정상 상태
Spring Boot ── DNS ── DB Endpoint ── Primary A
                                      │
                                   Standby C

장애 발생
Primary A 실패 → 장애 감지 → Standby C 승격
                         → Endpoint DNS 변경
                         → 애플리케이션 재조회
                         → 새 JDBC 연결
```

1. AWS가 DB 인스턴스, 스토리지, 네트워크 또는 AZ 이상을 감지한다.
2. 배포 유형에 맞는 Standby 또는 Replica가 새 Writer로 승격된다.
3. DB 엔드포인트의 DNS 레코드가 새 Writer를 가리키도록 변경된다.
4. 기존 연결은 실패하고 애플리케이션이 DNS를 다시 조회해 연결한다.
5. 실패 요청은 트랜잭션 결과와 멱등성을 판단한 뒤 제한적으로 재시도한다.

소요 시간은 엔진과 장애 유형, DNS 및 클라이언트 설정에 따라 달라지므로 고정된 수치를 가정하면 안 된다.

---

# 애플리케이션에는 어떤 일이 생기는가

Failover 중에는 짧은 연결 실패가 발생할 수 있다.

애플리케이션은 DB 연결을 다시 맺을 수 있어야 한다.

| 계층 | 발생 가능한 현상 | 필요한 대응 |
|---|---|---|
| DNS | 이전 주소 캐시 | 적절한 TTL과 재조회 |
| TCP/JDBC | 연결 끊김, 타임아웃 | 빠른 실패와 새 연결 |
| HikariCP | 죽은 연결 보관 | 검증과 keepalive 설정 |
| 트랜잭션 | 커밋 결과 불명확 | 상태 확인과 멱등성 |
| API | 5xx 또는 지연 | 제한된 재시도 |
| 비동기 작업 | 중간 실패 | 재처리와 중복 방지 |

---

# DNS가 중요한 이유

RDS와 Aurora 엔드포인트는 DNS 이름이며 Failover 시 이름은 유지되고 가리키는 대상이 바뀐다.

애플리케이션이 IP를 설정 파일에 저장하면 엔드포인트 전환 기능을 우회하게 된다.

JVM, 운영체제, 컨테이너 DNS, JDBC 드라이버에는 각기 캐시가 있을 수 있으므로 실제 배포 환경에서 재조회 시간을 측정해야 한다.

Java 보안 속성 `networkaddress.cache.ttl`은 DNS 캐시에 영향을 주지만 너무 짧게 하면 DNS 질의 부하가 늘 수 있다.

```text
networkaddress.cache.ttl=30
networkaddress.cache.negative.ttl=10
```

이 값은 예시이며 JDK, 런타임 환경과 RDS 권장 사항을 확인해 결정해야 한다.

---

# AWS 콘솔/CLI에서는

현재 상태와 Multi-AZ 여부, 엔드포인트를 확인한다.

```bash
aws rds describe-db-instances \
  --db-instance-identifier shop-prod-db \
  --query 'DBInstances[0].{Status:DBInstanceStatus,MultiAZ:MultiAZ,Endpoint:Endpoint.Address,AZ:AvailabilityZone}'
```

계획된 테스트에서는 RDS Multi-AZ DB 인스턴스 재부팅과 강제 Failover를 사용할 수 있다.

```bash
aws rds reboot-db-instance \
  --db-instance-identifier shop-prod-db \
  --force-failover
```

운영 테스트는 영향 범위, 중단 기준, 관찰 지표, 롤백 절차를 승인받은 뒤 수행해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

HikariCP는 연결 획득과 검증이 지나치게 오래 걸리지 않도록 설정하고 끊어진 연결을 교체할 수 있어야 한다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_ENDPOINT}:5432/shop
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      connection-timeout: 3000
      validation-timeout: 1000
      keepalive-time: 120000
      max-lifetime: 1800000
  jpa:
    open-in-view: false
```

`keepaliveTime`, `maxLifetime`, 네트워크 타임아웃은 DB와 인프라의 연결 정책을 고려해 부하 테스트로 조정한다.

재시도는 일시적 연결 예외에 한정하고 짧은 지수 백오프와 최대 횟수를 둔다.

```java
@Service
public class OrderCommandService {

    private final OrderRepository orderRepository;

    public OrderCommandService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public Order createOrder(String requestId, Order order) {
        return orderRepository.findByRequestId(requestId)
                .orElseGet(() ->
                        orderRepository.save(order.withRequestId(requestId)));
    }
}
```

`requestId`에는 유일 제약 조건을 두어 같은 요청이 재시도되어도 중복 생성을 막는다.

트랜잭션 전체를 무조건 재시도하면 외부 결제 호출이나 메시지 발행이 중복될 수 있다.

---

# 실무에서는 어떻게 사용할까

운영 전에 의도적인 Failover를 실행하고 DB 이벤트, API 오류율, Hikari 연결 상태, 복구 완료 시각을 기록한다.

사용자 요청은 멱등성 키를 받고 비동기 소비자는 이벤트 ID를 저장하여 재실행에 안전하게 만든다.

장애 대응자는 새 Writer 상태, 쓰기 가능 여부, 복제 상태, 데이터 정합성, 지연 작업을 순서대로 확인한다.

서비스 목표에는 AWS의 전환 시간뿐 아니라 애플리케이션이 정상 응답으로 돌아오는 전체 시간을 사용한다.

---

# 장애 사례와 주의할 점

커넥션 풀이 죽은 연결을 계속 빌려주면 Failover 완료 뒤에도 `Connection reset`과 타임아웃이 반복된다.

JVM이 이전 DNS 응답을 오래 캐시하면 새 Writer가 준비되어도 과거 주소로 접속한다.

커밋 응답 직전에 연결이 끊긴 주문을 실패로 단정하고 재실행하면 실제로는 커밋된 주문이 중복 생성될 수 있다.

모든 `SQLException`을 재시도하면 제약 조건 위반이나 잘못된 SQL 같은 영구 오류가 반복되어 장애를 키운다.

---

# 비용과 성능 고려사항

Failover를 가능하게 하는 Multi-AZ 또는 Aurora Replica는 추가 인스턴스, 스토리지, I/O와 데이터 전송 비용에 영향을 준다.

짧은 연결 검증 주기와 과도한 재시도는 정상 시 DB와 DNS에 불필요한 부하를 만들 수 있다.

가용성 비용은 장애 중 주문 손실과 복구 인력 비용을 포함한 업무 영향과 비교해야 한다.

정기 장애 테스트도 운영 비용이지만 검증되지 않은 고가용성 구성보다 예측 가능한 복구를 제공한다.

---

# 운영 점검 목록

- DB 인스턴스 IP가 아닌 엔드포인트를 사용하는가?
- JVM과 실행 환경의 DNS 캐시 동작을 확인했는가?
- 커넥션 풀이 죽은 연결을 빠르게 버리는가?
- 재시도 대상 예외와 최대 횟수가 제한되어 있는가?
- 주문과 결제 요청이 멱등한가?
- Failover 중 커밋 결과 불명확 상황을 처리하는가?
- 정기 테스트에서 전체 복구 시간을 측정하는가?

---

# 기억해야 할 내용

- Failover는 새 Primary로 전환하지만 무중단을 보장하지 않는다.
- 엔드포인트 DNS는 유지되지만 기존 JDBC 연결은 이동하지 않는다.
- DNS 캐시와 HikariCP의 죽은 연결이 복구 시간을 늘릴 수 있다.
- 재시도는 일시적 오류에만 제한하고 최대 횟수를 둔다.
- 쓰기 재시도에는 멱등성 키와 유일 제약 조건이 필요하다.
- 트랜잭션 커밋 결과가 불명확할 수 있다.
- 전체 애플리케이션을 포함한 정기 Failover 테스트가 필요하다.

---

# 다음 Chapter

다음 Chapter는 **Chapter 08. Part 5 Summary**이다.

RDS, Aurora, Multi-AZ, Read Replica, Backup, Restore, Failover를 하나의 운영 관점으로 연결하고 선택 기준을 정리한다.
