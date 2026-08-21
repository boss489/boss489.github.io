---
title: "Chapter 04. EventBridge"
permalink: /aws-backend/part-10/04-eventbridge/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. EventBridge
## 서비스 사이의 사건을 규칙으로 라우팅하는 버스

> **학습 목표**
>
> - EventBridge의 역할을 설명할 수 있다.
> - Event Bus와 Rule의 관계를 이해한다.
> - 이벤트 기반 아키텍처의 장단점을 설명할 수 있다.
> - 재시도, 멱등성, DLQ를 포함한 실패 처리를 설계할 수 있다.

---

# 왜 EventBridge가 필요한가

주문이 완료되면 알림 발송, 포인트 적립, 재고 반영, 정산 기록을 실행해야 한다고 가정해 보자.

주문 서비스가 네 시스템의 API를 직접 호출하면 대상 하나의 장애가 주문 응답을 늦추고 새로운 후처리를 추가할 때마다 주문 코드를 수정해야 한다.

각 대상의 주소, 인증 방식, 재시도 정책까지 주문 서비스가 알게 되면 서비스 사이의 결합도와 장애 범위가 커진다.

주문 서비스는 `OrderCompleted`라는 **발생한 사실만 발행**하고 각 소비자가 필요한 작업을 독립적으로 수행하는 구조가 필요하다.

EventBridge는 이벤트를 규칙으로 평가해 여러 Target으로 전달하여 생산자와 소비자의 직접 의존을 줄인다.

---

# EventBridge란?

EventBridge는 AWS 서비스, SaaS, 사용자 애플리케이션의 이벤트를 받아 Rule에 따라 Target으로 전달하는 관리형 이벤트 라우터이다.

---

# 구성 요소

| 구성 요소 | 역할 |
|---|---|
| Event | 발생한 사실과 메타데이터를 담은 JSON 문서이다. |
| Event Bus | 이벤트를 수신하고 Rule 평가의 경계를 제공한다. |
| Rule | Event Pattern 또는 Schedule로 이벤트를 선택한다. |
| Target | Lambda, SQS, SNS, Step Functions 등 전달 대상이다. |
| Archive | 이벤트를 보관하고 나중에 Replay할 수 있게 한다. |
| Schema | 이벤트 구조를 발견하고 코드 바인딩에 활용한다. |

---

# 동작 흐름

주문 완료 이벤트를 세 시스템으로 전달하는 흐름은 다음과 같다.

```
Order Service
  │ PutEvents: OrderCompleted
  ▼
Custom Event Bus
  │
  ├── Rule: detail-type=OrderCompleted
  │      ├──▶ SQS ──▶ 재고 Consumer
  │      ├──▶ Lambda ──▶ 알림 발송
  │      └──▶ Step Functions ──▶ 정산 흐름
  │
  └── 실패 전달 ──▶ DLQ
```

1. 주문 서비스는 `source`, `detail-type`, `detail`을 포함한 이벤트를 Event Bus에 발행한다.
2. EventBridge는 Bus에 연결된 각 Rule의 Event Pattern을 평가한다.
3. 일치한 Rule은 하나 이상의 Target으로 이벤트를 전달한다.
4. Target 호출이 실패하면 설정된 정책에 따라 재시도한다.
5. 재시도 후에도 실패한 이벤트는 DLQ로 보내 운영자가 원인 확인과 재처리를 할 수 있게 한다.

EventBridge의 전달은 중복 가능성을 고려해야 하므로 소비자는 같은 이벤트가 여러 번 도착해도 결과가 한 번만 반영되도록 설계한다.

---

# 이벤트 계약

좋은 이벤트는 명령이 아니라 이미 발생한 도메인 사실을 과거형 이름으로 표현한다.

```json
{
  "id": "5d3f6c8e-1d20-4b5f-9ed0-123456789abc",
  "source": "com.example.order",
  "detail-type": "OrderCompleted",
  "detail": {
    "eventVersion": 1,
    "orderId": "ORDER-100"
  }
}
```

계약을 변경할 때는 `eventVersion`을 두고 기존 소비자가 새 필드를 무시할 수 있도록 하위 호환성을 유지한다.

---

# EventBridge와 SNS와 SQS 비교

| 기준 | EventBridge | SNS | SQS |
|---|---|---|---|
| 핵심 역할 | 내용 기반 이벤트 라우팅 | Topic 기반 Fan-out | 메시지 버퍼와 소비 속도 조절 |
| 대상 선택 | Event Pattern | Topic 구독과 Filter | Queue Consumer |
| 메시지 보관 | Archive를 별도 구성한다. | 지속 Queue가 아니다. | 소비할 때까지 Queue에 보관한다. |
| 소비 방식 | Target으로 Push | 구독자에게 Push | Consumer가 Poll |
| 대표 용도 | 도메인 이벤트 통합 | 단순 알림 Fan-out | 비동기 작업 Queue |

서비스 간 이벤트 분류는 EventBridge, 단순한 Topic 방송은 SNS, 처리량 완충과 작업 보장은 SQS가 중심이며 함께 조합할 수 있다.

---

# 설정 예시

Custom Event Bus와 Rule을 생성하는 CLI 예시는 다음과 같다.

```bash
aws events create-event-bus \
  --name commerce-events

aws events put-rule \
  --name route-order-completed \
  --event-bus-name commerce-events \
  --event-pattern file://order-completed-pattern.json
```

Rule에서 사용할 Event Pattern은 필요한 필드만 선택해 작성한다.

```json
{
  "source": ["com.example.order"],
  "detail-type": ["OrderCompleted"],
  "detail": {
    "eventVersion": [1]
  }
}
```

---

# Spring Boot / Java에서는 어떻게 쓰는가

Spring Boot 생산자는 AWS SDK for Java v2의 `EventBridgeClient`로 도메인 이벤트를 발행할 수 있다.

```java
@Service
class OrderEventPublisher {
    private final EventBridgeClient eventBridgeClient;
    private final ObjectMapper objectMapper;
    OrderEventPublisher(
            EventBridgeClient eventBridgeClient, ObjectMapper objectMapper
    ) {
        this.eventBridgeClient = eventBridgeClient;
        this.objectMapper = objectMapper;
    }
    void publish(OrderCompleted event) throws JsonProcessingException {
        PutEventsRequestEntry entry = PutEventsRequestEntry.builder()
                .eventBusName("commerce-events")
                .source("com.example.order")
                .detailType("OrderCompleted")
                .detail(objectMapper.writeValueAsString(event))
                .build();
        PutEventsResponse response = eventBridgeClient.putEvents(
                PutEventsRequest.builder().entries(entry).build()
        );
        if (response.failedEntryCount() > 0) {
            throw new IllegalStateException("이벤트 발행 실패");
        }
    }
}

record OrderCompleted(int eventVersion, String eventId, String orderId) {}
```

데이터베이스 Commit과 이벤트 발행 사이의 이중 쓰기 문제는 Transactional Outbox 패턴과 별도 발행기로 해결하는 방식을 고려한다.

Lambda 소비자는 `RequestHandler<I, O>` 또는 Spring Cloud Function으로 구현할 수 있고 큰 Spring Boot 컨텍스트는 SnapStart, Provisioned Concurrency, 의존성 축소, GraalVM 네이티브 이미지를 검토한다.

**지속적으로 대량 이벤트를 순서대로 처리하거나 긴 실행과 세밀한 Backpressure가 필요하면 Lambda 직접 Target보다 SQS, Kafka, ECS Consumer가 적합할 수 있다.**

---

# 실무에서는 어떻게 사용할까

| 시나리오 | Event Pattern | Target |
|---|---|---|
| 주문 후처리 | `OrderCompleted` | 알림 Lambda, 정산 Step Functions |
| 보안 자동화 | 특정 AWS 상태 변경 | 대응 Lambda 또는 SNS |
| 스케줄 작업 | 예약 시각 | Lambda, ECS Task |
| 계정 간 통합 | 조직의 서비스 이벤트 | 중앙 Event Bus |
| 감사와 재처리 | 업무 이벤트 전체 | Archive와 Replay |

---

# 장애 사례

이벤트 재시도로 같은 주문 완료가 두 번 처리되어 쿠폰이 중복 지급되는 장애는 소비자의 멱등성 저장이 없을 때 발생한다.

Rule Pattern의 필드명이나 대소문자가 실제 이벤트와 다르면 오류 없이 Target 호출이 0건이 될 수 있으므로 대표 이벤트로 계약 테스트를 수행한다.

Target IAM 권한이 없거나 Lambda 리소스 정책이 누락되면 Rule은 일치하지만 전달이 실패한다.

DLQ를 구성하지 않으면 재시도가 끝난 실패 이벤트의 원문과 복구 대상을 찾기 어렵다.

주문 DB Commit은 성공했지만 EventBridge 발행 전에 프로세스가 종료되면 이벤트가 사라질 수 있으므로 중요한 이벤트는 Outbox 패턴을 검토한다.

---

# 주의할 점

이벤트 순서가 업무적으로 중요하면 순서가 뒤바뀌어도 검증 가능한 상태 버전이나 별도 순서 보장 채널을 설계한다.

Rule을 지나치게 넓게 작성하면 의도하지 않은 이벤트가 Target을 호출할 수 있으므로 `source`, `detail-type`, 버전을 함께 제한한다.

---

# 비용과 성능 고려사항

EventBridge 비용은 발행·전달하는 이벤트 수, Archive 저장과 Replay, 연결 대상의 사용량에 영향을 받는다.

Target이 Lambda이면 Lambda 요청 수와 실행 시간, SQS이면 요청과 저장, Step Functions이면 상태 전이 또는 실행 시간 비용이 추가된다.

---

# 기억해야 할 내용

- EventBridge는 이벤트를 Event Pattern으로 분류해 Target으로 전달한다.
- Event Bus, Rule, Target이 라우팅 구조의 핵심이다.
- 생산자는 발생한 사실을 발행하고 소비자의 구현을 알지 않아야 한다.
- EventBridge, SNS, SQS는 대체 관계가 아니라 역할에 따라 조합할 수 있다.
- 이벤트 전달은 중복될 수 있으므로 소비자는 멱등성을 가져야 한다.
- 중요한 이벤트는 DLQ, Archive, Replay, 알람과 함께 운영한다.
- DB 저장과 이벤트 발행의 이중 쓰기에는 Outbox 패턴을 고려한다.

---

# 다음 Chapter

다음 Chapter에서는 서버리스 애플리케이션의 데이터를 저장하는 **DynamoDB**를 학습한다.

Chapter 05는 [DynamoDB 상세 학습 목차](/aws-backend/part-10/05-dynamodb/) 아래에서 Key, Access Pattern, 동시성, 용량, 고급 모델링, Enhanced Client, 실전 문제의 7개 하위 챕터로 진행한다.

