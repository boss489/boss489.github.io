---
title: "Chapter 13. Interview Questions"
permalink: /aws-backend/part-16/13-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 13. Interview Questions
## Architecture Design 면접 질문

---

## 아키텍처 설계는 어디서 시작하는가?

기능 요구와 비기능 요구를 분리하고 트래픽, 지연, 가용성, RTO, RPO, 보안과 예산을 측정 가능한 가정으로 만든 뒤 제약과 대안을 비교한다.

## 쇼핑몰 전체 요청 흐름을 설명하라?

Route 53이 CloudFront로 연결하고 정적 파일은 OAC를 통해 S3로, API는 WAF와 ALB를 거쳐 여러 AZ의 Private ECS Task로 전달되며 Task는 Aurora, Redis와 메시징 서비스를 사용한다.

## 왜 모든 AWS 리소스를 한 줄로 직렬 연결하면 안 되는가?

S3 정적 파일 경로와 API 경로와 비동기 이벤트 경로가 서로 다르고 CloudWatch 같은 관측 도구는 요청의 직렬 의존성이 아니기 때문이다.

## Public Subnet과 Private Subnet을 어떻게 나누는가?

인터넷 진입점인 ALB와 외부 송신을 위한 NAT Gateway는 Public Subnet에 두고 ECS, Aurora와 Redis는 Private Subnet에 두며 서비스 통신에는 VPC Endpoint를 검토한다.

## REST API의 멱등성을 어떻게 보장하는가?

클라이언트가 보낸 Idempotency-Key를 사용자와 요청 해시 및 결과와 함께 저장하고 같은 요청은 기존 결과를 반환하며 다른 본문이면 충돌로 거부한다.

## Cursor 페이지네이션의 장점은 무엇인가?

정렬 키를 기준으로 다음 구간을 탐색하므로 깊은 Offset 비용을 피하고 데이터가 변경되는 동안 중복과 누락을 상대적으로 줄인다.

## 주문과 결제의 트랜잭션을 하나로 묶을 수 있는가?

로컬 주문 데이터는 DB 트랜잭션으로 묶을 수 있지만 외부 PG까지 원자적으로 묶기 어렵기 때문에 상태 머신, 멱등 호출과 보상 흐름이 필요하다.

## Aurora Reader를 언제 사용하는가?

상품 목록과 통계처럼 복제 지연을 허용하는 읽기에 사용하고 쓰기 직후 주문 확인처럼 Read-after-write가 필요한 조회는 Writer를 사용한다.

## Outbox Pattern은 무엇을 해결하는가?

업무 변경과 이벤트 레코드를 같은 DB 트랜잭션에 저장하여 Commit 후 발행 실패로 이벤트가 영구 유실되는 창을 줄인다.

## Aurora 대신 DynamoDB를 선택할 기준은 무엇인가?

접근 패턴이 사전에 명확하고 파티션 키 중심의 대규모 수평 확장이 중요하며 관계형 조인과 복잡한 트랜잭션 의존이 낮을 때 선택한다.

## Cache Aside의 정합성 문제를 어떻게 완화하는가?

Commit 후 캐시 삭제, 짧은 TTL, Jitter와 이벤트 기반 재무효화를 조합하고 업무상 오래된 값을 허용할 수 있는 시간을 정의한다.

## Cache Stampede를 어떻게 막는가?

TTL 만료를 Jitter로 분산하고 같은 키의 Miss를 요청 합치기나 안전한 Lock으로 제한하며 DB 우회량에 Backpressure를 적용한다.

## Redis 장애 시 어떻게 동작해야 하는가?

재고 원장 같은 핵심 정합성은 Redis에 의존하지 않고 상품 조회만 제한적으로 DB로 우회하며 Circuit Breaker와 Load Shedding으로 DB를 보호한다.

## MSK와 SNS/SQS와 EventBridge를 어떻게 비교하는가?

높은 처리량과 장기 Replay 및 Partition 순서는 MSK, 관리형 Queue와 Fan-out은 SNS/SQS, AWS 이벤트 통합과 규칙 라우팅은 EventBridge가 강하다.

## At-least-once 전달에서 필요한 것은 무엇인가?

중복 전달을 정상으로 보고 eventId 처리 기록이나 업무 고유 키와 원자적 상태 변경으로 소비자를 멱등하게 만든다.

## Exactly-once를 왜 과장하면 안 되는가?

Broker 내부 보장이 외부 DB와 결제 API를 포함한 전체 업무의 원자성을 뜻하지 않으므로 보장 범위와 실패 조건을 명시해야 한다.

## DLQ에 메시지가 들어가면 처리가 끝난 것인가?

아니며 원인을 수정하고 메시지 유효성을 확인한 뒤 속도를 제한해 Replay하고 결과를 대조하는 운영 절차가 필요하다.

## Spring Batch의 Chunk와 Checkpoint는 무엇인가?

일정 건수를 읽고 처리하고 쓴 뒤 Commit하여 진행 지점을 남기며 실패하면 마지막 성공 Checkpoint 이후부터 재시작하게 한다.

## 배치 중복 실행을 어떻게 막는가?

Job 이름과 businessDate를 고유 Parameter로 관리하고 동시 실행 Lock과 결과 테이블의 고유 제약 및 Upsert를 함께 적용한다.

## Presigned URL의 장점과 주의점은 무엇인가?

파일 본문이 ECS를 경유하지 않아 확장성이 좋아지지만 짧은 만료, 서버 생성 Key, 완료 후 크기 검증, CORS와 URL 유출 방지가 필요하다.

## S3 파일 처리에서 멱등성이 필요한 이유는 무엇인가?

S3 Event가 중복되거나 순서가 달라질 수 있으므로 Bucket, Key와 Version ID를 기준으로 같은 변환을 반복 반영하지 않아야 한다.

## CloudFront OAC는 왜 사용하는가?

S3 Bucket을 공개하지 않고 CloudFront만 원본 객체를 읽게 하여 사용자 접근 경로와 캐시 정책을 통제하기 위해 사용한다.

## ECS를 늘리면 처리량이 항상 증가하는가?

아니며 Aurora Writer, Connection, 외부 API와 Redis Hot Key가 병목이면 Task 증가는 오히려 의존성 부하를 키울 수 있다.

## Graceful Shutdown이 왜 중요한가?

배포와 Scale-in 때 새 요청 수신을 중단하고 진행 중 요청을 완료하여 연결 종료로 인한 오류와 부분 처리를 줄인다.

## 용량 모델은 어떻게 만드는가?

예시 RPS와 평균 서비스 시간으로 동시성을 추정하고 목표 사용률과 장애 시 여유를 반영한 뒤 정상, 피크와 급증 부하 테스트로 보정한다.

## Multi-AZ와 DR은 같은가?

아니며 Multi-AZ는 한 Region의 AZ 장애 대응이고 Region 전체 장애에는 별도 Region의 데이터와 인프라와 전환 절차가 필요하다.

## DR 전략 네 가지를 비교하라?

Backup/Restore에서 Pilot Light, Warm Standby, Multi-Site로 갈수록 일반적으로 RTO는 짧아질 수 있지만 비용과 데이터 일관성 및 운영 복잡도는 커진다.

## RTO와 RPO의 차이는 무엇인가?

RTO는 허용 가능한 복구 시간이고 RPO는 허용 가능한 데이터 손실 시점이며 업무 데이터 종류마다 다르게 정한다.

## 백업 검증은 어떻게 하는가?

백업 성공 여부만 보지 않고 격리 환경에 Restore하여 무결성을 검사하고 애플리케이션을 기동해 실제 복구 시간을 측정한다.

## Well-Architected 여섯 Pillar는 무엇인가?

운영 우수성, 보안, 신뢰성, 성능 효율성, 비용 최적화와 지속 가능성이다.

## Threat Model은 무엇을 포함하는가?

보호할 자산, 사용자와 시스템 Actor, 신뢰 경계, 데이터 흐름, 가능한 위협과 완화책 및 잔여 위험을 포함한다.

## ADR은 왜 필요한가?

당시 상황과 대안과 선택 이유와 결과 및 재검토 조건을 남겨 이후 변경에서 같은 논쟁을 반복하지 않게 한다.

## 과도한 설계를 어떻게 피하는가?

현재 측정된 병목과 합의한 목표를 해결하는 최소 구조를 선택하고 확장 신호와 교체 경계를 남기며 복잡도의 운영 비용도 평가한다.

# 면접 답변 원칙

- 숫자는 확정값이 아니라 예시 가정인지 밝히고 측정 방법을 설명한다.
- 무중단이나 Exactly-once 같은 절대 표현 대신 보장 범위와 실패 조건을 말한다.
- 요구사항에서 제약과 대안과 결정과 검증으로 이어지는 순서로 답한다.
- 비용과 보안과 운영 책임을 기능 설계와 함께 언급한다.

# 다음 Part

다음 단계인 [Part 17. Final Project](/aws-backend/part-17/)에서는 지금까지 내린 설계 결정을 실제 Spring Boot 서비스와 AWS 인프라로 구현한다.
