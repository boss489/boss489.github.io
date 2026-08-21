---
title: "Chapter 07. X-Ray Distributed Tracing"
permalink: /aws-backend/part-11/07-xray-distributed-tracing/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. X-Ray Distributed Tracing
## 분산 요청의 경로와 병목을 추적하는 방법

> **학습 목표**
>
> - Trace, Segment, Subsegment의 관계를 설명할 수 있다.
> - Trace ID와 context 전파 원리를 이해할 수 있다.
> - Sampling과 비동기 경계를 설계할 수 있다.
> - OpenTelemetry와 ADOT의 역할을 설명할 수 있다.

---

# 왜 필요한가

주문 요청 일부가 4초 이상 걸리지만 각 서비스 평균은 정상이라고 가정한다.
주문 API, 재고 API, 결제 API 로그의 시계를 맞춰 수동으로 찾는 데 시간이 오래 걸린다.
하나의 Trace ID가 전파되면 호출 경로와 각 구간의 시간을 한눈에 비교할 수 있다.
메시지 큐를 지나는 비동기 처리는 producer와 consumer 문맥 연결을 명시해야 한다.
샘플링되지 않은 요청도 오류 메트릭과 로그로 감지할 수 있어야 한다.

관측 데이터는 많이 모으는 것이 목적이 아니라 고객 영향과 다음 행동을 빠르게 결정하기 위한 근거이다.

---

# X-Ray Distributed Tracing이란?

AWS X-Ray는 분산 요청의 Trace를 Segment와 Subsegment로 수집하고 서비스 맵과 지연 분석을 제공하는 추적 서비스이다.

신호의 이름과 책임 경계를 먼저 정해야 여러 팀이 같은 데이터를 같은 의미로 해석할 수 있다.

---

# 동작 흐름

```
Client trace context
       │
       ▼
[ALB Segment]
       │ trace ID
       ▼
[Order Segment]
  ├─ [RDS Subsegment]
  ├─ [Redis Subsegment]
  └─ [Payment HTTP Subsegment]
              │
              └─▶ X-Ray / ADOT Collector
```

흐름은 다음 순서로 이해한다.

1. 주문 요청 일부가 4초 이상 걸리지만 각 서비스 평균은 정상이라고 가정한다.
2. 주문 API, 재고 API, 결제 API 로그의 시계를 맞춰 수동으로 찾는 데 시간이 오래 걸린다.
3. 하나의 Trace ID가 전파되면 호출 경로와 각 구간의 시간을 한눈에 비교할 수 있다.
4. 메시지 큐를 지나는 비동기 처리는 producer와 consumer 문맥 연결을 명시해야 한다.

각 단계는 실패할 수 있으므로 수집 누락과 전달 지연도 별도 상태로 다룬다.

---

# 핵심 개념

| 개념 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| Trace | 요청 전체 경로 | 하나의 주문 호출 |
| Segment | 서비스 처리 단위 | order-api |
| Subsegment | 내부·외부 작업 | SQL 또는 HTTP |
| Trace ID | 경로 상관 식별자 | 헤더로 전파 |
| Sampling | 수집 요청 선택 | 비용과 대표성 조정 |
| Service Map | 의존성과 오류 시각화 | 병목 서비스 탐색 |

개념을 독립적으로 외우기보다 하나의 장애 질문에 어떤 개념이 답하는지 연결한다.

---

# AWS CLI 설정과 조회

다음 명령은 이름과 ARN을 실습 환경에 맞게 바꿔 실행한다.

```bash
aws xray get-service-graph --start-time 2026-08-05T00:00:00Z --end-time 2026-08-05T01:00:00Z
aws xray get-trace-summaries --start-time 2026-08-05T00:00:00Z --end-time 2026-08-05T01:00:00Z --filter-expression 'service("order-api") AND responsetime > 2'
aws xray batch-get-traces --trace-ids 1-6891abcd-0123456789abcdef01234567
```

CLI 실행 역할에는 조회와 변경에 필요한 최소 권한만 부여한다.

운영 변경은 콘솔에서 임의로 수행하지 않고 IaC와 리뷰 절차로 재현 가능하게 관리한다.

---

# Spring Boot 3.x와 Java 17 연동

OpenTelemetry Java Agent로 Spring MVC, HTTP client, JDBC 계측을 코드 변경 없이 시작할 수 있다.
ADOT Collector는 OTLP 데이터를 받아 X-Ray 등 백엔드로 라우팅하는 AWS 배포판 구성 요소이다.
W3C traceparent와 필요한 X-Ray 형식을 경계에서 전파한다.
Executor와 메시지 consumer에서는 context를 캡처하고 복원해 비동기 연결을 유지한다.
Span attribute에 orderId 원문이나 개인정보를 무분별하게 넣지 않는다.

생성자 주입을 사용하면 관측 구현을 테스트 대역으로 교체하고 의존성을 명확히 할 수 있다.

```java
@Service
public class CheckoutObservationService {
    private final MeterRegistry meterRegistry;

    public CheckoutObservationService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void recordSuccess(String paymentProvider) {
        meterRegistry.counter("checkout.success",
                "provider", paymentProvider).increment();
    }
}
```

provider 값은 허용 목록으로 제한하고 주문 번호나 사용자 ID는 tag로 사용하지 않는다.

---

# 실무 운영

- 오류와 느린 요청의 샘플 우선순위를 높이는 규칙을 검토한다.
- 서비스 이름, 환경, 버전 속성을 표준화해 서비스 맵 분리를 방지한다.
- Trace에서 관련 로그로 이동할 수 있도록 traceId를 MDC에 기록한다.
- Collector의 큐, 재시도, 메모리 제한도 별도 모니터링한다.
- 추적 백엔드 장애가 고객 요청을 실패시키지 않도록 비동기 내보내기를 사용한다.

## 운영 점검 순서

1. 고객 영향과 시작 시각을 확인한다.
2. 최근 배포와 구성 변경을 확인한다.
3. 서비스 수준 신호에서 영향 범위를 좁힌다.
4. 로그와 트레이스로 실패 문맥을 찾는다.
5. 완화 조치를 수행하고 지표 회복을 검증한다.
6. 타임라인과 재발 방지 작업을 기록한다.

---

# 장애 사례

## 사례 1

HTTP 클라이언트가 trace header를 제거해 서비스마다 별도 Trace가 생성되었다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 2

CompletableFuture 작업에서 context가 유실되어 비동기 구간이 루트 Span으로 보였다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 3

모든 요청을 수집해 저장량과 애플리케이션 오버헤드가 증가했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 4

Span을 종료하지 않아 잘못된 긴 지연과 메모리 사용이 발생했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

---

# 비용과 성능 고려사항

- 기록·검색·보관하는 Trace 수와 Collector 자원이 비용과 성능에 영향을 준다.
- 샘플링은 트래픽, 오류 희소성, 조사 요구를 기준으로 조정한다.
- 속성과 이벤트를 과도하게 추가하면 네트워크와 저장량이 증가한다.
- Agent와 Collector 적용 전후 CPU, 메모리, 지연을 부하 테스트한다.

정확한 단가는 Region과 시점에 따라 달라지므로 공식 요금 페이지와 실제 사용량으로 확인한다.

비용 최적화는 데이터를 무조건 버리는 일이 아니라 질문에 답하지 못하는 데이터를 줄이는 일이다.

---

# 기억할 내용

- Trace, Segment, Subsegment의 관계를 설명할 수 있다.
- Trace ID와 context 전파 원리를 이해할 수 있다.
- Sampling과 비동기 경계를 설계할 수 있다.
- OpenTelemetry와 ADOT의 역할을 설명할 수 있다.
- 로그·메트릭·트레이스는 공통 서비스와 환경 정보로 연결한다.
- 개인정보와 인증 정보는 관측 데이터에 남기지 않는다.
- 수집량과 카디널리티와 보존 기간을 운영 정책으로 관리한다.
- 알람은 소유자와 runbook이 있을 때 행동 가능한 신호가 된다.

---

# 다음 Chapter

다음 Chapter에서는 [Spring Boot Observability](/aws-backend/part-11/08-spring-boot-observability/)를 학습한다.

현재 Chapter의 신호를 다음 단계의 운영 판단과 연결한다.
