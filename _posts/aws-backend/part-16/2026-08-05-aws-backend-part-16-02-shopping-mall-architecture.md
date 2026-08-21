---
title: "Chapter 02. Shopping Mall Architecture"
permalink: /aws-backend/part-16/02-shopping-mall-architecture/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Shopping Mall Architecture
## 실제 요청 경로와 네트워크 경계를 반영한 전체 설계

> **학습 목표**
>
> - 쇼핑몰의 동기 요청과 파일 경로를 분리한다.
> - Public Subnet과 Private Subnet의 책임을 구분한다.
> - Multi-AZ 배치와 관측 가능성을 설계에 포함한다.
> - NAT Gateway와 VPC Endpoint의 비용 및 보안을 비교한다.

---

# 요구사항과 실패 시나리오

사용자는 Route 53이 반환한 CloudFront 배포 도메인으로 접속한다.

CloudFront는 정적 객체는 S3 Origin으로, API 동작은 WAF를 거쳐 ALB Origin으로 전달한다.

인터넷에 노출되는 것은 ALB이며 ECS Task에는 Public IP를 부여하지 않는다.

ALB와 NAT Gateway는 Public Subnet에, ECS와 Aurora와 Redis는 Private Subnet에 둔다.

ECS Service는 최소 두 AZ에 Task를 분산하고 ALB Health Check를 통과한 Task만 등록한다.

피크 시간에 하위 시스템 하나가 느려졌을 때 전체 주문 경로가 함께 멈추는 구조인지 먼저 질문한다.

네트워크 재시도와 중복 메시지와 AZ 장애를 정상적인 운영 조건으로 간주하고 설계를 검증한다.

---

# 설계 원칙

요구사항을 측정 가능한 지표로 바꾸고 제약과 가정을 명시한다.

동기 경로는 사용자 응답에 꼭 필요한 작업으로 줄이고 나머지는 비동기로 분리한다.

상태는 명시적인 원본 저장소에 두고 캐시와 실행 노드는 교체 가능하게 만든다.

실패를 숨기기보다 Timeout과 격리와 재처리와 관측 지점을 설계한다.

가장 복잡한 기술보다 현재 규모에서 운영할 수 있고 진화 가능한 결정을 선택한다.

---

# 아키텍처

```text
Users
  | DNS
Route 53
  |
CloudFront -- OAC --> Private S3
  | API behavior + WAF
Public Subnets: ALB(AZ-a, AZ-b)  NAT per AZ
  |
Private App: ECS-a <----> ECS-b
  +--> Redis Multi-AZ / Aurora Writer+Reader
  +--> S3 via VPC Endpoint
CloudWatch Logs/Metrics/Traces watches every path
```

다이어그램의 화살표는 실제 데이터 경로를 나타내며 관측 도구는 업무 저장소 뒤에 직렬로 놓인 구성 요소가 아니다.

---

# 선택지 비교

| 기준 | 선택 A | 선택 B | 결정 기준 |
|---|---|---|---|
| 구간 | 대안 | 선택 | 이유 |
| API 진입 | API Gateway | ALB | ECS HTTP 서비스와 경로 라우팅 |
| 실행 | EC2 직접 운영 | ECS Fargate | 운영 부담 절감 |
| 외부 통신 | NAT만 사용 | Endpoint 병행 | 비용과 공격면 감소 |
| 정적 파일 | App 경유 | CloudFront와 S3 | 대역폭 분리 |

비교표의 결정은 영구 결론이 아니라 현재 요구와 팀의 운영 역량에 기반한 선택이다.

---

# 동작 흐름과 결정

1. 사용자는 Route 53이 반환한 CloudFront 배포 도메인으로 접속한다.

2. CloudFront는 정적 객체는 S3 Origin으로, API 동작은 WAF를 거쳐 ALB Origin으로 전달한다.

3. 인터넷에 노출되는 것은 ALB이며 ECS Task에는 Public IP를 부여하지 않는다.

4. ALB와 NAT Gateway는 Public Subnet에, ECS와 Aurora와 Redis는 Private Subnet에 둔다.

5. ECS Service는 최소 두 AZ에 Task를 분산하고 ALB Health Check를 통과한 Task만 등록한다.

6. Aurora Writer는 쓰기를 처리하고 Reader는 지연 허용 조회를 분산한다.

7. Redis는 상품 캐시가 실패해도 DB로 제한적으로 우회할 수 있게 한다.

8. S3 Gateway Endpoint는 S3 통신의 NAT 경유를 줄이고 Interface Endpoint는 서비스별 비용을 비교한다.

9. CloudWatch 지표와 로그, X-Ray 또는 OpenTelemetry Trace에 요청 식별자를 연결한다.

10. AZ 장애 시 남은 AZ 용량이 피크의 필수 기능을 감당하는지 검증한다.

결정 기록에는 선택하지 않은 대안과 다시 검토할 조건도 함께 남긴다.

---

# 구현 예

```yaml
server:
  shutdown: graceful
management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

예제 설정과 수치는 동작 원리를 보여 주기 위한 가정이며 운영값은 측정과 부하 테스트로 확정한다.

---

# 실무 Trade-off

가용성을 높이는 복제와 대기 자원은 비용을 늘리므로 업무 중요도별로 등급을 나눈다.

강한 정합성은 안전한 결정을 돕지만 지연과 결합도를 높일 수 있어 필요한 경계에만 적용한다.

관리형 서비스는 운영 부담을 줄이지만 서비스 한도와 종속성과 비용 모델을 이해해야 한다.

비동기화는 장애 격리와 흡수력을 높이지만 상태 추적과 중복 처리와 재처리 운영이 필요하다.

초기 설계는 단순하게 시작하되 지표와 인터페이스를 남겨 병목이 확인되면 교체할 수 있게 한다.

---

# 장애, 보안과 비용

장애 시나리오는 증상, 탐지 지표, 자동 완화, 수동 Runbook과 복구 확인 순서로 작성한다.

IAM Role에는 필요한 Action과 Resource만 허용하고 장기 Access Key를 애플리케이션에 저장하지 않는다.

전송 구간 TLS와 저장 데이터 KMS 암호화를 적용하고 개인정보 접근은 감사 가능하게 남긴다.

재시도는 지수 Backoff와 Jitter와 횟수 제한을 사용하여 장애 중 부하 증폭을 막는다.

비용은 컴퓨팅 시간, 저장량, 요청 수, 데이터 전송과 고정 네트워크 비용으로 나눠 추적한다.

특정 달러 가격은 Region과 시점과 약정에 따라 달라지므로 확정값 대신 AWS Pricing Calculator와 실제 청구로 검증한다.

---

# 검증 질문

- 이 결정이 해결하려는 구체적인 요구사항과 제약은 무엇인가.
- 한 AZ 또는 외부 의존성이 실패하면 사용자가 어떤 응답을 받는가.
- 중복 요청과 지연된 이벤트와 부분 성공을 어떻게 식별하고 복구하는가.
- 가장 먼저 포화되는 자원과 이를 알려 주는 선행 지표는 무엇인가.
- 보안 경계와 데이터 소유자와 최소 권한의 증거는 무엇인가.
- 예상 비용의 가장 큰 항목과 트래픽 증가에 따른 기울기는 무엇인가.
- 설계 가정을 어떤 부하 테스트와 장애 훈련으로 반증할 수 있는가.

---

# 기억해야 할 내용

- 쇼핑몰의 동기 요청과 파일 경로를 분리한다.
- Public Subnet과 Private Subnet의 책임을 구분한다.
- Multi-AZ 배치와 관측 가능성을 설계에 포함한다.
- NAT Gateway와 VPC Endpoint의 비용 및 보안을 비교한다.
- 요구사항에서 제약과 대안을 거쳐 결정을 내리고 지표와 훈련으로 검증한다.
- 실패와 비용과 운영 책임이 없는 아키텍처 다이어그램은 완성된 설계가 아니다.

---

# 다음 Chapter

다음 Chapter에서는 [REST API Design](/aws-backend/part-16/03-rest-api-design/)를 학습한다.

앞 장의 결정을 다음 장의 더 구체적인 경계와 운영 절차로 연결한다.
