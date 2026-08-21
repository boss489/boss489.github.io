---
title: "Chapter 06. Backup and Restore"
permalink: /aws-backend/part-05/06-backup-restore/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Backup and Restore
## 장애 이후 데이터를 되돌리는 기준

> **학습 목표**
>
> - Backup과 Restore의 차이를 설명할 수 있다.
> - RPO와 RTO의 의미를 이해한다.
> - 운영 DB 백업 정책의 기본 요소를 설명할 수 있다.
> - 특정 시점 복구와 스냅샷 복구 절차를 설계할 수 있다.

---

# 왜 백업과 복구가 필요한가

쇼핑몰 운영자가 조건을 잘못 입력한 배치로 최근 한 시간의 주문을 삭제했다고 가정해 보자.

DB 인스턴스와 AZ가 모두 정상이고 Multi-AZ Standby도 있지만 삭제 트랜잭션은 정상 변경으로 인식되어 Standby에도 복제된다.

Read Replica 역시 같은 변경을 뒤따라 적용하므로 가용성 복제만으로는 과거 데이터를 되찾을 수 없다.

랜섬웨어, 애플리케이션 버그, 스키마 배포 오류, 계정 탈취, 지역 단위 재해에도 독립된 복구 지점이 필요하다.

백업은 복구 가능한 데이터를 보존하고 복구는 그 데이터를 실제 서비스 가능한 DB로 되돌리는 과정이다.

---

# Backup이란?

Backup은 장애나 실수에 대비해 데이터를 복구 가능한 형태로 보관하는 것이다.

RDS는 자동 백업과 스냅샷을 제공한다.

자동 백업은 설정한 보관 기간 동안 특정 시점 복구(Point-in-Time Recovery)를 지원하기 위해 백업과 트랜잭션 로그를 관리한다.

수동 DB 스냅샷은 사용자가 명시적으로 생성한 복구 지점이며 삭제하기 전까지 보관할 수 있지만 보관 비용과 수명 주기를 관리해야 한다.

---

# Restore란?

Restore는 백업을 이용해 데이터베이스를 복구하는 작업이다.

복구는 보통 기존 DB를 되돌리는 것이 아니라 새로운 DB 인스턴스를 만드는 방식으로 진행된다.

따라서 Restore가 완료되어도 애플리케이션의 연결 대상 변경, 보안 그룹, 파라미터, 데이터 검증 작업이 남는다.

```
운영 DB
  │
  ├── 자동 백업 + 트랜잭션 로그 ── 특정 시점 복구
  │
  └── 수동 스냅샷 ─────────────── 스냅샷 시점 복구
                                      │
                               새 DB 인스턴스
                                      │
                         검증 → 연결 전환 → 서비스 재개
```

---

# 자동 백업과 수동 스냅샷 비교

| 기준 | 자동 백업 | 수동 스냅샷 |
|---|---|---|
| 생성 | RDS가 정책에 따라 관리 | 사용자 또는 자동화가 생성 |
| 복구 단위 | 보관 기간 내 특정 시점 | 스냅샷 생성 시점 |
| 보관 | 설정한 보관 기간 | 명시적으로 삭제할 때까지 |
| 용도 | 일상적인 PITR | 배포 전, 장기 보관, 명시적 기준점 |
| 운영 책임 | 보관 기간과 백업 창 설정 | 태그, 보존, 삭제 정책 관리 |

자동 백업과 스냅샷 모두 실제 복원 및 검증을 해보기 전에는 목표 RTO를 만족하는지 알 수 없다.

---

# RPO와 RTO

| 개념 | 질문 | 쇼핑몰 예시 |
|---|---|---|
| RPO | 얼마만큼의 데이터 손실을 허용하는가 | 최근 주문을 어느 시점까지 복구해야 하는가 |
| RTO | 얼마 동안 서비스 중단을 허용하는가 | 주문 API를 언제까지 재개해야 하는가 |

RPO가 짧을수록 더 촘촘한 복구 지점과 로그 보존이 필요하고 RTO가 짧을수록 자동화와 사전 준비가 중요하다.

서비스의 모든 데이터가 같은 목표를 가질 필요는 없으며 주문, 상품 통계, 개발 데이터의 중요도에 따라 등급을 나눌 수 있다.

---

# 복구 흐름

1. 장애 변경이 시작된 시각과 영향 범위를 확인한다.
2. 허용 가능한 마지막 정상 시점을 결정한다.
3. 자동 백업 또는 스냅샷에서 새 DB를 복원한다.
4. DB Subnet Group, 보안 그룹, 파라미터와 권한을 확인한다.
5. 주문 수, 금액 합계, 참조 무결성, 핵심 쿼리로 데이터를 검증한다.
6. Spring Boot의 연결 대상을 새 엔드포인트로 전환한다.
7. 누락된 이벤트나 외부 결제 결과를 재처리하고 모니터링한다.

복구 중에도 원본 DB를 즉시 삭제하지 말고 원인 분석과 데이터 비교에 필요한 기간 동안 격리해 보존한다.

---

# AWS 콘솔/CLI에서는

자동 백업 설정과 최근 복구 가능 시점을 확인한다.

```bash
aws rds describe-db-instances \
  --db-instance-identifier shop-prod-db \
  --query 'DBInstances[0].{Retention:BackupRetentionPeriod,Window:PreferredBackupWindow,Earliest:EarliestRestorableTime,Latest:LatestRestorableTime}'
```

수동 스냅샷은 중요한 스키마 변경 전에 명시적으로 만들 수 있다.

```bash
aws rds create-db-snapshot \
  --db-instance-identifier shop-prod-db \
  --db-snapshot-identifier shop-prod-before-migration
```

복원 명령에는 새 DB 식별자를 사용하며 네트워크와 파라미터 설정을 운영 기준에 맞게 지정한다.

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier shop-prod-db \
  --target-db-instance-identifier shop-prod-restore \
  --restore-time 2026-08-21T00:30:00Z
```

실제 복구 가능 시각과 지원 옵션은 대상 엔진 및 구성에서 조회한 값으로 확인해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

애플리케이션은 복원된 새 엔드포인트를 환경 변수로 주입받아 재배포하거나 안전한 설정 전환 절차를 거친다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/shop
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      connection-timeout: 3000
      validation-timeout: 1000
  jpa:
    hibernate:
      ddl-auto: validate
```

복구 데이터 검증을 위해 운영 코드의 읽기 전용 서비스나 별도 검증 잡을 재사용할 수 있다.

```java
@Service
public class RestoreValidationService {

    private final OrderRepository orderRepository;

    public RestoreValidationService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional(readOnly = true)
    public RestoreValidationResult validate(Instant from, Instant to) {
        long orderCount = orderRepository.countByCreatedAtBetween(from, to);
        BigDecimal totalAmount =
                orderRepository.sumAmountByCreatedAtBetween(from, to);
        return new RestoreValidationResult(orderCount, totalAmount);
    }
}
```

검증 쿼리는 단순 접속 성공을 넘어 업무적으로 데이터가 올바른지 확인해야 한다.

---

# 실무에서는 어떻게 사용할까

주문 DB는 자동 백업 보관 기간과 백업 창을 정하고 대규모 배포 전에는 별도 스냅샷을 생성할 수 있다.

스냅샷에 환경, 서비스, 소유 팀, 만료 예정일 태그를 붙여 장기 방치와 실수 삭제를 줄인다.

정기 복구 훈련에서 별도 격리 환경으로 복원하고 소요 시간, 수동 단계, 실패 원인을 기록한다.

계정 또는 리전 수준 재해가 요구사항에 포함되면 교차 계정·교차 리전 스냅샷 복사와 KMS 키 접근 정책도 검토한다.

---

# 장애 사례와 주의할 점

백업 성공 이벤트만 확인하고 복구 테스트를 하지 않으면 손상, 권한, 설정 누락을 실제 사고 때 처음 발견한다.

복원된 DB가 기본 보안 그룹이나 다른 파라미터 그룹을 사용하면 애플리케이션 접속 또는 동작이 운영 DB와 달라질 수 있다.

PITR 시점을 삭제 직후로 선택하면 잘못된 트랜잭션까지 포함되므로 장애 시작 시각을 정확히 찾아야 한다.

복구 후 외부 결제 시스템과 DB 상태가 어긋날 수 있으므로 결제사 원장과 이벤트 재처리 절차가 필요하다.

---

# 비용과 성능 고려사항

백업 비용은 보관량, 보존 기간, 수동 스냅샷, 리전 간 복사, 복구 테스트용 인스턴스와 데이터 전송에 영향을 받는다.

보관 기간을 무조건 길게 설정하기보다 규제와 업무 RPO에 맞춘 계층별 정책과 만료 자동화를 사용한다.

백업 창의 I/O 영향과 복구 중 새 인스턴스 생성 시간을 운영 부하에서 측정해야 한다.

비용 절감 목적으로 복구 훈련을 생략하면 실제 장애의 RTO를 예측할 수 없다.

---

# 기억해야 할 내용

- Multi-AZ와 Read Replica는 과거 데이터 복구를 위한 백업이 아니다.
- 자동 백업은 보관 기간 내 특정 시점 복구를 지원한다.
- 수동 스냅샷은 명시적 복구 지점이며 수명 주기 관리가 필요하다.
- Restore는 보통 기존 DB를 덮어쓰지 않고 새 DB를 만든다.
- RPO는 허용 데이터 손실, RTO는 허용 복구 시간이다.
- 복구 후에는 네트워크, 설정, 업무 데이터, 외부 시스템을 검증해야 한다.
- 백업 정책은 정기적인 복구 리허설까지 포함해야 한다.

---

# 다음 Chapter

다음 Chapter에서는 Primary 장애 시 대기 노드로 역할을 전환하는 [Failover](/aws-backend/part-05/07-failover/)를 학습한다.

DNS 변경과 커넥션 풀, 재시도, 트랜잭션이 복구 시간에 어떤 영향을 주는지 살펴본다.