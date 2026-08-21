---
title: "Chapter 07. Batch Architecture"
permalink: /aws-backend/part-16/07-batch-architecture/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Batch Architecture
## 재시작 가능하고 중복되지 않는 정기 작업

> **학습 목표**
>
> - Spring Batch의 Job, Step, Chunk와 Checkpoint를 이해한다.
> - ECS Task와 AWS Batch의 실행 모델을 비교한다.
> - EventBridge Scheduler와 Step Functions의 책임을 나눈다.
> - Retry, Skip과 중복 실행 방지 정책을 설계한다.

---

# 요구사항과 실패 시나리오

매일 정산 Job은 주문 API와 자원을 경쟁하지 않도록 별도 ECS Task로 실행한다.

EventBridge Scheduler는 정해진 시각에 RunTask 또는 Step Functions 실행을 시작한다.

컴퓨팅 요구가 크고 Queue와 자원 최적화가 중요하면 AWS Batch를 비교한다.

여러 단계와 분기와 대기와 외부 작업 조정은 Step Functions로 가시화할 수 있다.

Spring Batch Chunk는 일정 수를 읽고 처리하고 쓰며 Commit 지점을 Checkpoint로 남긴다.

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
EventBridge Schedule
         |
Step Functions(optional)
    +----+----+
    |         |
ECS RunTask  AWS Batch
    |
Spring Batch Job
Reader -> Processor -> Writer
       chunk commit/checkpoint
Failed item -> quarantine / alarm
```

다이어그램의 화살표는 실제 데이터 경로를 나타내며 관측 도구는 업무 저장소 뒤에 직렬로 놓인 구성 요소가 아니다.

---

# 선택지 비교

| 기준 | 선택 A | 선택 B | 결정 기준 |
|---|---|---|---|
| 선택 | 강점 | 약점 | 적용 |
| ECS RunTask | 기존 이미지 재사용 | 스케줄 관리 필요 | 일반 정산 |
| AWS Batch | Queue와 컴퓨팅 최적화 | 구성 증가 | 대규모 병렬 |
| Step Functions | 오케스트레이션 가시성 | 상태 전환 비용 | 다단계 흐름 |
| 상시 Service | 즉시 시작 | 유휴 비용 | 빈번한 작업 |

비교표의 결정은 영구 결론이 아니라 현재 요구와 팀의 운영 역량에 기반한 선택이다.

---

# 동작 흐름과 결정

1. 매일 정산 Job은 주문 API와 자원을 경쟁하지 않도록 별도 ECS Task로 실행한다.

2. EventBridge Scheduler는 정해진 시각에 RunTask 또는 Step Functions 실행을 시작한다.

3. 컴퓨팅 요구가 크고 Queue와 자원 최적화가 중요하면 AWS Batch를 비교한다.

4. 여러 단계와 분기와 대기와 외부 작업 조정은 Step Functions로 가시화할 수 있다.

5. Spring Batch Chunk는 일정 수를 읽고 처리하고 쓰며 Commit 지점을 Checkpoint로 남긴다.

6. Chunk 크기가 너무 크면 Rollback 비용이 커지고 너무 작으면 Commit 오버헤드가 커진다.

7. 일시적 네트워크 오류만 제한적으로 Retry하고 잘못된 업무 데이터는 Skip 후 격리한다.

8. 같은 businessDate에 동일 Job이 동시에 실행되지 않도록 고유 Job Parameter와 잠금을 사용한다.

9. 재실행 시 결과 테이블의 업무 고유 키와 Upsert로 중복 반영을 방지한다.

10. 대량 작업은 API DB의 Connection Pool과 IOPS를 고갈시키지 않도록 속도 제한과 실행 창을 둔다.

결정 기록에는 선택하지 않은 대안과 다시 검토할 조건도 함께 남긴다.

---

# 구현 예

```java
@Bean
Step settlementStep(JobRepository repository,
                    PlatformTransactionManager transactionManager) {
    return new StepBuilder("settlementStep", repository)
            .<OrderRow, SettlementRow>chunk(500, transactionManager)
            .reader(orderReader())
            .processor(settlementProcessor())
            .writer(settlementWriter())
            .faultTolerant().retryLimit(3).retry(TransientDataAccessException.class)
            .skipLimit(20).skip(ValidationException.class)
            .build();
}
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

- Spring Batch의 Job, Step, Chunk와 Checkpoint를 이해한다.
- ECS Task와 AWS Batch의 실행 모델을 비교한다.
- EventBridge Scheduler와 Step Functions의 책임을 나눈다.
- Retry, Skip과 중복 실행 방지 정책을 설계한다.
- 요구사항에서 제약과 대안을 거쳐 결정을 내리고 지표와 훈련으로 검증한다.
- 실패와 비용과 운영 책임이 없는 아키텍처 다이어그램은 완성된 설계가 아니다.

---

# 다음 Chapter

다음 Chapter에서는 [File Delivery Architecture](/aws-backend/part-16/08-file-delivery-architecture/)를 학습한다.

앞 장의 결정을 다음 장의 더 구체적인 경계와 운영 절차로 연결한다.
