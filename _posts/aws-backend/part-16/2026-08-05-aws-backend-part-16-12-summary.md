---
title: "Chapter 12. Part 16 Summary"
permalink: /aws-backend/part-16/12-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---
# Chapter 12. Part 16 Summary
## 요구사항에서 운영 검증까지 연결하는 아키텍처 요약
> **학습 목표**
>
> - Part 16의 주요 설계 결정을 하나의 흐름으로 설명한다.
> - 선택한 AWS 서비스보다 선택 근거와 검증 방법을 먼저 말한다.
> - 장애와 보안과 비용을 출시 조건에 포함한다.
---
# 설계의 출발점
기능 요구사항은 회원, 상품, 장바구니, 주문, 결제, 파일과 운영자 흐름을 정의한다.
비기능 요구사항은 트래픽 범위, 데이터 증가량, 지연 시간, 가용성, RTO, RPO, 보안과 예산을 측정 가능하게 정의한다.
모든 임의 트래픽 수치와 비용은 예시 가정으로 표시하고 운영 계측과 부하 테스트로 보정한다.
현재 요구보다 훨씬 큰 시스템을 미리 만들지 않고 변경 신호와 진화 경로를 함께 남긴다.
---
# 전체 요청 흐름
```text
Users
  |
Route 53
  |
CloudFront -- OAC --> Private S3
  |
WAF -> Public ALB
          |
   Private ECS Tasks across AZs
      +---+---+---+
      |   |   |   |
   Aurora Redis SQS/SNS or MSK
      |
CloudWatch Logs/Metrics/Traces
```
CloudFront는 정적 파일과 API Behavior를 분리하고 ALB는 정상 ECS Target에만 요청을 전달한다.
ECS, Aurora와 Redis는 Private Subnet에 배치하고 S3 접근에는 VPC Endpoint를 검토한다.
NAT Gateway는 외부 통신에 필요할 수 있지만 AZ별 가용성과 처리 비용을 함께 계산한다.
---
# API와 데이터
REST API는 리소스 URI, HTTP 메서드와 상태 코드의 의미를 보존한다.
주문 생성은 Idempotency-Key로 네트워크 재시도 중 중복 주문을 막는다.
목록 조회는 안정적인 정렬 키를 포함한 Cursor 페이지네이션을 사용한다.
Bean Validation과 업무 검증을 분리하고 오류 응답에는 안정적인 code와 traceId를 제공한다.
OpenAPI를 계약으로 관리하고 파괴적 변경에만 새 주요 API 버전을 도입한다.
주문과 주문 항목과 Outbox는 하나의 Aurora 로컬 트랜잭션으로 저장한다.
외부 결제 호출은 DB 트랜잭션 밖에서 상태 머신과 보상 흐름으로 처리한다.
쓰기 직후 조회는 Writer를 사용하고 지연 허용 조회만 Reader로 분리한다.
인덱스는 실제 필터와 정렬과 Cardinality 및 실행 계획을 근거로 선택한다.

DynamoDB는 조인보다 예측 가능한 키 접근과 수평 확장이 핵심일 때 선택한다.

---

# 캐시와 이벤트

Cache Aside는 Redis Miss에서 Aurora를 읽고 TTL과 Jitter를 적용해 저장한다.

변경 후 무효화와 짧은 TTL을 함께 사용해 DB와 캐시 사이의 불일치 창을 제한한다.

Stampede에는 요청 합치기, Lock, 사전 갱신과 Backpressure를 상황에 맞게 적용한다.

Redis 장애 시 필수 조회만 제한적으로 DB로 우회하고 과부하가 시작되면 기능을 축소한다.

이벤트 Broker는 처리량과 Replay와 순서와 운영 역량을 기준으로 MSK, SNS/SQS와 EventBridge를 비교한다.

At-least-once 전달에서는 eventId와 업무 고유 키로 소비자를 멱등하게 만든다.

순서는 전역이 아니라 orderId 같은 Aggregate 범위의 Partition Key로 제한한다.

Outbox와 CDC는 DB Commit과 이벤트 발행 사이의 유실 창을 줄인다.

DLQ는 실패 메시지의 종착지가 아니라 원인 수정과 통제된 Replay의 출발점이다.

Exactly-once는 외부 시스템 전체의 원자성을 자동 보장한다는 뜻으로 사용하지 않는다.

---

# 배치와 파일

EventBridge Scheduler는 ECS RunTask 또는 Step Functions를 정해진 시각에 시작한다.

Spring Batch는 Chunk Commit과 Checkpoint로 실패 지점부터 재시작한다.

일시적 오류만 Retry하고 업무 오류는 Skip과 격리 후 담당자가 확인한다.

businessDate와 Job 이름을 고유하게 관리하고 결과 쓰기도 멱등하게 만든다.

대규모 병렬 컴퓨팅과 Queue 최적화가 필요하면 AWS Batch를 검토한다.

파일은 Presigned URL로 Browser에서 S3에 직접 업로드한다.

S3 Event는 SQS로 버퍼링하고 Worker가 검증과 악성 코드 검사와 변환을 수행한다.

CloudFront OAC는 사용자가 Private S3 Origin에 직접 접근하지 못하게 한다.

불변 Object Key와 Versioning은 캐시 무효화와 복구를 단순하게 만든다.

---

# 확장성과 가용성

ECS Task는 상태 없이 여러 AZ로 수평 확장한다.

ALB Health Check와 Readiness와 Graceful Shutdown을 배포 및 Scale-in 정책과 맞춘다.

Task 수 증가가 Aurora Connection과 Writer 병목을 악화시키지 않는지 확인한다.

Timeout, Bulkhead, 제한된 Retry와 Load Shedding으로 느린 의존성의 전파를 막는다.

용량 모델은 RPS와 서비스 시간과 목표 사용률로 시작하고 피크 부하 테스트로 교정한다.

Multi-AZ는 Region 내부 가용성이고 재해 복구를 대신하지 않는다.

---

# 재해 복구와 리뷰

Backup/Restore, Pilot Light, Warm Standby와 Multi-Site는 비용과 RTO/RPO가 서로 다르다.

Region 간 복제는 비동기 지연과 전환 시 미복제 데이터 가능성을 포함한다.

백업 성공 알람만으로 끝내지 않고 격리 환경 Restore와 Region Failover 훈련을 수행한다.

Well-Architected의 운영 우수성, 보안, 신뢰성, 성능 효율성, 비용 최적화와 지속 가능성을 모두 검토한다.

Threat Model은 자산과 신뢰 경계와 공격 경로를 보여 준다.

Failure Mode Review는 구성 요소 실패의 영향과 탐지와 완화와 소유자를 기록한다.

ADR은 상황, 대안, 결정, 결과와 재검토 조건을 남긴다.

운영 준비 검토에는 Dashboard, Alarm, Runbook, On-call, 배포와 Rollback 및 용량 한도가 포함된다.

---

# 최종 검증 질문

- 요구사항마다 설계 결정과 측정 지표가 연결되어 있는가.
- 단일 AZ와 Region과 외부 결제 장애의 사용자 영향을 설명할 수 있는가.
- 주문과 결제와 이벤트의 중복 및 부분 성공을 복구할 수 있는가.
- 보안 경계와 최소 권한과 감사 증거가 준비되어 있는가.
- 비용 추정의 가정과 실제 청구를 비교할 수 있는가.
- 부하 테스트와 Restore 및 Failover 훈련 결과가 목표를 충족하는가.

---

# 다음 Chapter

다음 Chapter에서는 [Interview Questions](/aws-backend/part-16/13-interview/)로 설계 근거를 짧고 정확하게 설명하는 연습을 한다.
