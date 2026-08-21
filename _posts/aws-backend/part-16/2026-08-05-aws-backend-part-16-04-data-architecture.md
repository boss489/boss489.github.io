---
title: "Chapter 04. Data Architecture"
permalink: /aws-backend/part-16/04-data-architecture/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Data Architecture
## 주문과 결제의 경계 및 읽기 확장을 함께 설계

> **학습 목표**
>
> - 주문과 결제의 트랜잭션 경계를 명확히 한다.
> - Aurora Writer와 Reader의 일관성 차이를 다룬다.
> - 쿼리 패턴에서 인덱스를 역으로 설계한다.
> - Outbox와 DynamoDB 선택 기준을 설명한다.

---

# 요구사항과 실패 시나리오

주문 생성은 주문과 주문 항목과 결제 요청 기록을 하나의 로컬 트랜잭션으로 저장한다.

외부 PG 호출을 DB 트랜잭션 안에서 기다리지 않고 결제 상태 머신으로 분리한다.

결제 승인과 주문 확정이 동시에 원자적일 수 없으므로 보상과 재처리를 설계한다.

쓰기 직후 주문 상세은 Reader 복제 지연을 피하려고 Writer에서 읽는다.

상품 목록과 통계는 약간의 지연을 허용할 때 Reader Endpoint를 사용한다.

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
Order API
  | local transaction
Aurora Writer
  +-- orders / order_lines / payments
  +-- outbox
           | CDC or publisher
           v
       Event Broker
Aurora Reader <--- delay-tolerant query
Writer <----------- read-after-write
DynamoDB <--------- key-value workload only
```

다이어그램의 화살표는 실제 데이터 경로를 나타내며 관측 도구는 업무 저장소 뒤에 직렬로 놓인 구성 요소가 아니다.

---

# 선택지 비교

| 기준 | 선택 A | 선택 B | 결정 기준 |
|---|---|---|---|
| 기준 | Aurora | DynamoDB | 결정 신호 |
| 트랜잭션 | 관계와 조인 강점 | 항목 중심 | 주문 원장에는 Aurora |
| 확장 | Writer 병목 가능 | 파티션 확장 | 키 접근량 |
| 쿼리 | 유연한 SQL | 사전 접근 패턴 | 변경 가능성 |
| 일관성 | 강한 읽기 가능 | 옵션별 차이 | 업무 요구 |

비교표의 결정은 영구 결론이 아니라 현재 요구와 팀의 운영 역량에 기반한 선택이다.

---

# 동작 흐름과 결정

1. 주문 생성은 주문과 주문 항목과 결제 요청 기록을 하나의 로컬 트랜잭션으로 저장한다.

2. 외부 PG 호출을 DB 트랜잭션 안에서 기다리지 않고 결제 상태 머신으로 분리한다.

3. 결제 승인과 주문 확정이 동시에 원자적일 수 없으므로 보상과 재처리를 설계한다.

4. 쓰기 직후 주문 상세은 Reader 복제 지연을 피하려고 Writer에서 읽는다.

5. 상품 목록과 통계는 약간의 지연을 허용할 때 Reader Endpoint를 사용한다.

6. Reader Endpoint는 연결을 분산할 뿐 쿼리마다 부하를 분산하지 않는다.

7. 인덱스는 `customer_id, created_at, id`처럼 실제 필터와 정렬 순서를 반영한다.

8. 중복 인덱스는 쓰기 비용과 저장량을 늘리므로 실행 계획과 느린 쿼리로 검증한다.

9. Outbox 행을 업무 변경과 같은 트랜잭션에 저장하고 Publisher가 이벤트를 전달한다.

10. DynamoDB는 접근 패턴이 명확하고 파티션 키로 수평 확장이 가능할 때 검토하며 조인이 핵심이면 Aurora가 적합하다.

결정 기록에는 선택하지 않은 대안과 다시 검토할 조건도 함께 남긴다.

---

# 구현 예

```java
@Transactional
public OrderId placeOrder(PlaceOrder command) {
    Order order = Order.place(command.customerId(), command.lines());
    orderRepository.save(order);
    paymentRepository.save(Payment.requested(order.id(), command.amount()));
    outboxRepository.save(OutboxEvent.of("Order", order.id(), "OrderPlaced"));
    return order.id();
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

- 주문과 결제의 트랜잭션 경계를 명확히 한다.
- Aurora Writer와 Reader의 일관성 차이를 다룬다.
- 쿼리 패턴에서 인덱스를 역으로 설계한다.
- Outbox와 DynamoDB 선택 기준을 설명한다.
- 요구사항에서 제약과 대안을 거쳐 결정을 내리고 지표와 훈련으로 검증한다.
- 실패와 비용과 운영 책임이 없는 아키텍처 다이어그램은 완성된 설계가 아니다.

---

# 다음 Chapter

다음 Chapter에서는 [Cache Consistency](/aws-backend/part-16/05-cache-consistency/)를 학습한다.

앞 장의 결정을 다음 장의 더 구체적인 경계와 운영 절차로 연결한다.
