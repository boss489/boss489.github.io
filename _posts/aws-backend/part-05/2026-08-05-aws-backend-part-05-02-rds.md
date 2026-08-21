---
title: "Chapter 02. RDS"
permalink: /aws-backend/part-05/02-rds/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. RDS
## 관계형 데이터베이스 관리형 서비스

> **학습 목표**
>
> - RDS의 역할을 설명할 수 있다.
> - 직접 설치한 DB와 RDS의 차이를 이해한다.
> - RDS 운영에서 사용자가 책임질 부분을 설명할 수 있다.
> - Spring Boot의 데이터소스와 커넥션 풀을 안전하게 설정할 수 있다.

---

# 왜 RDS가 필요한가

쇼핑몰 팀이 EC2에 PostgreSQL을 직접 설치하면 주문 기능뿐 아니라 운영체제, DB 프로세스, 디스크, 백업, 패치까지 관리해야 한다.

디스크나 백업 작업이 실패해도 팀이 직접 감지해야 하며 야간 장애에는 담당자가 복구 절차를 수행해야 한다.

Spring Boot API 개발에 집중해야 하는 팀에게 이런 반복 작업은 큰 운영 부담이다.

RDS는 데이터베이스 인프라 작업을 AWS가 관리하여 팀이 스키마, 쿼리, 데이터 품질에 집중하게 한다.

다만 관리형이라는 말은 AWS가 느린 쿼리나 잘못된 권한, 데이터 삭제까지 해결한다는 뜻이 아니다.

---

# RDS란?

RDS(Relational Database Service)는 관계형 데이터베이스를 관리형으로 제공하는 AWS 서비스다.

지원 엔진에는 MySQL, PostgreSQL, MariaDB, Oracle Database, Microsoft SQL Server가 있으며 Aurora도 RDS 서비스 체계에서 관리된다.

사용자는 DB 인스턴스 클래스, 엔진 버전, 스토리지, 네트워크, 고가용성, 백업 정책을 선택한다.

AWS는 선택한 설정을 기반으로 DB 인스턴스를 프로비저닝하고 상태 지표와 자동화된 운영 기능을 제공한다.

---

# 구조

RDS 인스턴스는 선택한 VPC의 DB Subnet Group과 보안 그룹에 연결되며 애플리케이션은 엔드포인트 DNS로 접속한다.

```
Private Subnet A             Private Subnet C
┌─────────────────┐         ┌─────────────────┐
│ Spring Boot API │         │ Spring Boot API │
└────────┬────────┘         └────────┬────────┘
         └────────────┬──────────────┘
                      │ TCP 5432
               DB Security Group
                      │
              RDS DB Endpoint
                      │
             ┌────────▼────────┐
             │ PostgreSQL RDS  │
             └────────┬────────┘
                      │
              Managed Storage
```

DB Subnet Group은 RDS가 사용할 수 있는 여러 AZ의 서브넷 집합이며 실제 접근 허용 여부는 보안 그룹이 결정한다.

엔드포인트는 인스턴스 교체나 장애 전환 이후 대상이 바뀔 수 있으므로 IP 주소를 고정하면 안 된다.

---

# RDS가 대신 하는 것과 사용자가 하는 것

| 영역 | AWS가 관리하는 부분 | 사용자가 결정하는 부분 |
|---|---|---|
| 인프라 | 서버 프로비저닝, 하드웨어 유지보수 | 인스턴스 클래스와 배치 |
| 스토리지 | 볼륨 연결과 관리 기능 | 유형, 용량, 성능 설정 |
| 백업 | 자동 백업과 스냅샷 기능 | 보관 기간과 복구 검증 |
| 패치 | 유지보수 기능과 일정 창 | 적용 정책과 영향 검토 |
| 가용성 | Multi-AZ 기능 | 활성화 여부와 장애 테스트 |
| 모니터링 | CloudWatch 지표 제공 | 경보 기준과 대응 절차 |
| 데이터 | 저장 엔진 실행 | 스키마, 인덱스, 쿼리, 품질 |
| 보안 | 서비스 수준 격리 기능 | 계정, 권한, 암호화, SG 규칙 |

RDS는 운영 부담을 줄이지만 데이터베이스 설계와 데이터에 대한 책임은 사용자에게 남는다.

---

# 내부 동작 원리

애플리케이션이 엔드포인트를 DNS로 조회하면 현재 DB 인스턴스에 연결되는 주소를 얻는다.

Spring Boot의 HikariCP는 TCP 연결과 인증을 수행한 뒤 연결을 풀에 보관하고 요청마다 재사용한다.

JPA가 SQL을 실행하면 DB 엔진이 쿼리 계획을 세우고 버퍼와 스토리지를 사용해 결과를 처리한다.

자동 백업이 활성화되어 있으면 RDS는 백업과 트랜잭션 로그를 관리하여 보관 기간 안의 특정 시점 복구를 지원한다.

Multi-AZ를 사용하면 변경 사항이 다른 AZ의 대기 환경으로 복제되고 장애 시 엔드포인트가 새 Primary를 가리키도록 전환된다.

---

# 주요 엔진 선택 기준

| 엔진 | 고려할 상황 | 확인할 점 |
|---|---|---|
| PostgreSQL | 표준 SQL, 확장 기능, 복잡한 쿼리 | 확장 모듈 지원 범위 |
| MySQL | 넓은 생태계와 익숙한 운영 경험 | 버전별 기능과 복제 특성 |
| MariaDB | MariaDB 호환성이 필수인 시스템 | MySQL과의 차이 |
| Oracle | 기존 Oracle 워크로드 | 라이선스와 기능 제약 |
| SQL Server | Microsoft 생태계 연동 | 에디션과 라이선스 |

엔진 선택은 단순 성능 비교보다 기존 스키마, 드라이버, 운영 기술, 라이선스, 마이그레이션 비용을 기준으로 해야 한다.

---

# AWS 콘솔/CLI에서는

RDS 생성 시 운영 환경은 `Public access`를 끄고 Private Subnet과 애플리케이션 보안 그룹을 기준으로 접근을 제한한다.

암호화, 자동 백업 보관 기간, 삭제 방지, 유지보수 기간, 모니터링 기능도 함께 결정한다.

```bash
aws rds describe-db-instances \
  --db-instance-identifier shop-prod-db \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Engine:Engine,MultiAZ:MultiAZ,Storage:AllocatedStorage,Endpoint:Endpoint.Address}'
```

자격 증명은 CLI 인자나 Git에 남기지 않고 Secrets Manager 등으로 관리한다.

---

# Spring Boot에서는 어떻게 쓰는가

Spring Boot 3.x에서는 엔진에 맞는 JDBC 드라이버와 `spring-boot-starter-data-jpa`를 사용한다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/shop
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      pool-name: shop-pool
      maximum-pool-size: 15
      minimum-idle: 5
      connection-timeout: 3000
      validation-timeout: 1000
      max-lifetime: 1800000
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
```

운영 환경에서 `ddl-auto: create` 또는 `update`에 의존하지 않고 Flyway나 Liquibase로 스키마 변경 이력을 관리한다.

커넥션 풀의 최대 크기는 `애플리케이션 인스턴스 수 × maximum-pool-size`로 계산하고 관리 도구와 배치 연결도 여유분에 포함한다.

```java
@Service
public class ProductQueryService {

    private final ProductRepository productRepository;

    public ProductQueryService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional(readOnly = true)
    public Product findProduct(long productId) {
        return productRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId));
    }
}
```

`open-in-view: false`로 요청 전체에 DB 연결이 불필요하게 유지되는 상황을 줄이고 서비스 계층에서 조회 범위를 명확히 한다.

---

# 실무에서는 어떻게 사용할까

운영 RDS는 Private Subnet에 배치하고 DB 보안 그룹의 인바운드는 Spring Boot 애플리케이션 보안 그룹에서 오는 DB 포트만 허용한다.

개발, 스테이징, 운영 데이터베이스를 분리하고 운영 계정은 최소 권한과 감사 가능한 절차로 사용한다.

CloudWatch에서 CPU, 여유 메모리, DB 연결 수, 스토리지 여유, 읽기·쓰기 지연, IOPS를 경보로 관리한다.

느린 쿼리는 애플리케이션 로그만 보지 말고 DB의 쿼리 통계와 실행 계획을 함께 분석한다.

---

# 장애 사례와 주의할 점

Auto Scaling으로 API 서버가 늘면서 각 서버의 HikariCP가 최대 연결을 열어 RDS의 연결 한도를 소진할 수 있다.

보안 그룹을 IP 하나로 허용하면 서버 교체 후 주소가 바뀌어 DB 접속이 실패할 수 있으므로 보안 그룹 참조를 사용한다.

자동 백업만 믿고 복구 시간을 측정하지 않으면 실제 사고에서 새 인스턴스 생성, DNS 변경, 검증에 예상보다 오래 걸릴 수 있다.

마이너 버전 패치나 인스턴스 변경은 연결 단절을 일으킬 수 있으므로 유지보수 창과 재연결 동작을 검증해야 한다.

---

# 비용과 성능 고려사항

RDS 비용은 인스턴스 실행 시간, 스토리지 용량과 성능, I/O, 백업 보관량, 데이터 전송, Multi-AZ 및 Replica 구성에 영향을 받는다.

큰 인스턴스로 올리기 전에 인덱스 누락, N+1 쿼리, 불필요한 전체 조회, 잠금 경합을 확인해야 한다.

개발 환경은 가용성 요구가 낮을 수 있지만 운영 환경의 Multi-AZ와 백업을 비용만으로 제거하면 장애 비용이 더 커질 수 있다.

---

# 기억해야 할 내용

- RDS는 관계형 데이터베이스의 반복 인프라 운영을 관리형으로 제공한다.
- 스키마, 인덱스, 쿼리, 계정과 데이터 품질은 사용자의 책임이다.
- 운영 DB는 Private Subnet과 제한된 보안 그룹을 사용한다.
- 애플리케이션은 DB 인스턴스 IP가 아니라 엔드포인트 DNS를 사용한다.
- 전체 커넥션 수는 모든 애플리케이션 인스턴스를 합산해 설계한다.
- 자동 백업 설정과 실제 복구 가능성은 별개의 문제이다.
- 성능 문제는 인스턴스 크기와 쿼리 구조를 함께 분석해야 한다.

---

# 다음 Chapter

다음 Chapter에서는 AWS가 설계한 MySQL/PostgreSQL 호환 엔진인 [Aurora](/aws-backend/part-05/03-aurora/)를 학습한다.

일반 RDS 엔진과 달리 컴퓨팅과 분산 스토리지를 분리한 구조가 가용성과 읽기 확장에 어떤 영향을 주는지 살펴본다.
