---
title: "Chapter 08. Spring Boot Observability"
permalink: /aws-backend/part-11/08-spring-boot-observability/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Spring Boot Observability
## 로그·메트릭·트레이스를 하나의 요청으로 연결하는 방법

> **학습 목표**
>
> - Actuator와 Micrometer의 역할을 설명할 수 있다.
> - CloudWatch Registry와 Managed Prometheus 경로를 구분할 수 있다.
> - MDC correlation ID와 Trace ID를 연결할 수 있다.
> - OpenTelemetry Agent와 ADOT Collector를 구성할 수 있다.

---

# 왜 필요한가

고객이 주문 버튼을 눌렀지만 오류 화면을 봤다고 가정한다.
운영자는 correlation ID로 JSON 로그를 찾고 같은 traceId의 분산 경로를 연다.
결제 실패율 메트릭으로 전체 영향과 시작 시점을 확인한다.
세 신호가 서비스, 환경, 버전, 요청 문맥으로 연결되면 추측 대신 증거로 대응한다.
관측 기능은 장애 중에도 애플리케이션을 추가로 불안정하게 만들지 않아야 한다.

관측 데이터는 많이 모으는 것이 목적이 아니라 고객 영향과 다음 행동을 빠르게 결정하기 위한 근거이다.

---

# Spring Boot Observability이란?

Spring Boot Observability는 Actuator, Micrometer, 로깅 문맥, OpenTelemetry를 이용해 애플리케이션의 로그·메트릭·트레이스를 일관된 속성으로 생성하고 연결하는 설계이다.

신호의 이름과 책임 경계를 먼저 정해야 여러 팀이 같은 데이터를 같은 의미로 해석할 수 있다.

---

# 동작 흐름

```
HTTP Request: X-Correlation-Id
            │
            ▼
Spring Boot 3.x
  ├─ Logback JSON + MDC ──▶ CloudWatch Logs
  ├─ Micrometer Registry ─▶ CloudWatch Metrics
  ├─ /actuator/prometheus ─▶ AMP Collector path
  └─ OTel Java Agent ─────▶ ADOT ─▶ X-Ray
            │
     traceId / service / env / version
```

흐름은 다음 순서로 이해한다.

1. 고객이 주문 버튼을 눌렀지만 오류 화면을 봤다고 가정한다.
2. 운영자는 correlation ID로 JSON 로그를 찾고 같은 traceId의 분산 경로를 연다.
3. 결제 실패율 메트릭으로 전체 영향과 시작 시점을 확인한다.
4. 세 신호가 서비스, 환경, 버전, 요청 문맥으로 연결되면 추측 대신 증거로 대응한다.

각 단계는 실패할 수 있으므로 수집 누락과 전달 지연도 별도 상태로 다룬다.

---

# 핵심 개념

| 개념 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| Actuator | 운영 endpoint와 자동 구성 | health, metrics, prometheus |
| Micrometer | 벤더 중립 메트릭 facade | Counter, Timer, Gauge |
| CloudWatch Registry | CloudWatch 직접 발행 | 단순 AWS 구성 |
| Prometheus Endpoint | 스크레이프 형식 노출 | AMP 수집 경로 |
| MDC | 로그 문맥 저장 | correlationId, traceId |
| OpenTelemetry Agent | 자동 trace 계측 | Spring, JDBC, HTTP |

개념을 독립적으로 외우기보다 하나의 장애 질문에 어떤 개념이 답하는지 연결한다.

---

# AWS CLI 설정과 조회

다음 명령은 이름과 ARN을 실습 환경에 맞게 바꿔 실행한다.

```bash
aws cloudwatch list-metrics --namespace shopping-mall --metric-name http.server.requests
aws logs filter-log-events --log-group-name /ecs/order-api --filter-pattern '{ $.correlationId = "req-123" }'
curl -fsS http://localhost:8080/actuator/health
curl -fsS http://localhost:8080/actuator/prometheus
```

CLI 실행 역할에는 조회와 변경에 필요한 최소 권한만 부여한다.

운영 변경은 콘솔에서 임의로 수행하지 않고 IaC와 리뷰 절차로 재현 가능하게 관리한다.

---

# Spring Boot 3.x와 Java 17 연동

`spring-boot-starter-actuator`와 필요한 Micrometer Registry를 명시적으로 추가한다.
`/actuator/prometheus`는 Amazon Managed Service for Prometheus가 직접 공개 인터넷으로 읽는 주소가 아니라 안전한 collector 또는 agent가 scrape하는 endpoint로 설계한다.
CloudWatch MeterRegistry는 직접 push 방식이며 Prometheus scrape 방식과 중복 발행하지 않도록 목적을 정한다.
CorrelationIdFilter는 생성자 주입된 정책 객체를 사용하고 MDC를 finally에서 정리한다.
OpenTelemetry Java Agent 설정은 코드와 분리하고 service.name, deployment.environment, service.version을 지정한다.

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

- Actuator endpoint는 필요한 항목만 노출하고 관리 네트워크와 인증으로 보호한다.
- health의 liveness와 readiness를 구분해 외부 의존성 장애가 재시작 폭풍을 만들지 않게 한다.
- 공통 tag와 resource attribute는 낮은 카디널리티로 통제한다.
- 로그 패턴에 traceId와 spanId를 포함해 trace에서 로그로 이동한다.
- Telemetry exporter 실패율과 큐 적체도 자체 지표로 감시한다.

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

모든 Actuator endpoint를 공개해 환경 정보와 설정이 노출되었다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 2

Prometheus endpoint를 만들기만 하고 scraper를 배치하지 않아 메트릭이 AMP에 저장되지 않았다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 3

MDC를 정리하지 않아 다른 고객 요청의 correlation ID가 로그에 섞였다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 4

CloudWatch Registry와 Prometheus를 동시에 켜 같은 메트릭을 중복 수집했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

---

# 비용과 성능 고려사항

- Custom Metric 시계열, 로그 수집·보관, Trace 샘플, AMP 수집·저장·조회가 과금 요소이다.
- 공통 tag 추가 전에 생성될 시계열 수를 계산한다.
- 운영에 필요 없는 Actuator 메트릭은 deny filter로 제외한다.
- 관측 신호의 수집 실패가 업무 요청 지연으로 전파되지 않도록 batch와 timeout을 제한한다.

정확한 단가는 Region과 시점에 따라 달라지므로 공식 요금 페이지와 실제 사용량으로 확인한다.

비용 최적화는 데이터를 무조건 버리는 일이 아니라 질문에 답하지 못하는 데이터를 줄이는 일이다.

---

# 기억할 내용

- Actuator와 Micrometer의 역할을 설명할 수 있다.
- CloudWatch Registry와 Managed Prometheus 경로를 구분할 수 있다.
- MDC correlation ID와 Trace ID를 연결할 수 있다.
- OpenTelemetry Agent와 ADOT Collector를 구성할 수 있다.
- 로그·메트릭·트레이스는 공통 서비스와 환경 정보로 연결한다.
- 개인정보와 인증 정보는 관측 데이터에 남기지 않는다.
- 수집량과 카디널리티와 보존 기간을 운영 정책으로 관리한다.
- 알람은 소유자와 runbook이 있을 때 행동 가능한 신호가 된다.

---

# 다음 Chapter

다음 Chapter에서는 [Summary](/aws-backend/part-11/09-summary/)를 학습한다.

현재 Chapter의 신호를 다음 단계의 운영 판단과 연결한다.
