---
title: "Chapter 02. CloudWatch Logs"
permalink: /aws-backend/part-11/02-cloudwatch-logs/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. CloudWatch Logs
## 사건의 문맥을 안전하게 기록하고 검색하는 방법

> **학습 목표**
>
> - Log Group과 Log Stream의 관계를 설명할 수 있다.
> - JSON 구조화 로그와 보존 기간을 설계할 수 있다.
> - Logs Insights로 장애 문맥을 검색할 수 있다.
> - Subscription Filter와 Metric Filter의 차이를 구분할 수 있다.

---

# 왜 필요한가

주문 20260805-1001이 결제 완료 후 취소되었다는 문의가 들어왔다고 가정한다.
평문 로그가 여러 줄로 흩어져 있으면 한 요청의 처리 결과를 연결하기 어렵다.
JSON 필드와 requestId가 있으면 서비스와 Task가 달라도 사건을 추적할 수 있다.
카드 번호와 인증 토큰을 로그에 남기면 조사 편의보다 큰 보안 사고가 된다.
로그 수집 단계부터 마스킹과 최소 수집 원칙을 적용해야 한다.

관측 데이터는 많이 모으는 것이 목적이 아니라 고객 영향과 다음 행동을 빠르게 결정하기 위한 근거이다.

---

# CloudWatch Logs이란?

CloudWatch Logs는 애플리케이션과 AWS 서비스의 로그 이벤트를 Log Group과 Log Stream으로 수집·보관·검색하는 관리형 서비스이다.

신호의 이름과 책임 경계를 먼저 정해야 여러 팀이 같은 데이터를 같은 의미로 해석할 수 있다.

---

# 동작 흐름

```
ECS Task A stdout ──▶ Stream ecs/order/a
                         │
ECS Task B stdout ──▶ Stream ecs/order/b
                         │
                         ▼
                 Log Group /ecs/order-api
                  ├─ retention 30 days
                  ├─ Logs Insights
                  ├─ Metric Filter
                  └─ Subscription ──▶ Firehose/Lambda
```

흐름은 다음 순서로 이해한다.

1. 주문 20260805-1001이 결제 완료 후 취소되었다는 문의가 들어왔다고 가정한다.
2. 평문 로그가 여러 줄로 흩어져 있으면 한 요청의 처리 결과를 연결하기 어렵다.
3. JSON 필드와 requestId가 있으면 서비스와 Task가 달라도 사건을 추적할 수 있다.
4. 카드 번호와 인증 토큰을 로그에 남기면 조사 편의보다 큰 보안 사고가 된다.

각 단계는 실패할 수 있으므로 수집 누락과 전달 지연도 별도 상태로 다룬다.

---

# 핵심 개념

| 개념 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| Log Group | 보존·권한·암호화 정책 단위 | /ecs/order-api |
| Log Stream | 발생 원천별 이벤트 흐름 | ECS Task 또는 인스턴스 |
| Log Event | timestamp와 message | JSON 한 건 |
| Retention | 자동 삭제 기간 | 환경별 보존 정책 |
| Metric Filter | 패턴을 메트릭으로 변환 | ERROR 건수 |
| Subscription Filter | 실시간 외부 전달 | 분석 파이프라인 |

개념을 독립적으로 외우기보다 하나의 장애 질문에 어떤 개념이 답하는지 연결한다.

---

# AWS CLI 설정과 조회

다음 명령은 이름과 ARN을 실습 환경에 맞게 바꿔 실행한다.

```bash
aws logs create-log-group --log-group-name /ecs/order-api
aws logs put-retention-policy --log-group-name /ecs/order-api --retention-in-days 30
aws logs start-query --log-group-name /ecs/order-api --start-time 1785888000 --end-time 1785974400 --query-string 'fields @timestamp, requestId, level, message | filter level = "ERROR" | sort @timestamp desc | limit 50'
```

CLI 실행 역할에는 조회와 변경에 필요한 최소 권한만 부여한다.

운영 변경은 콘솔에서 임의로 수행하지 않고 IaC와 리뷰 절차로 재현 가능하게 관리한다.

---

# Spring Boot 3.x와 Java 17 연동

Logback JSON Encoder를 사용해 timestamp, level, service, requestId를 필드로 출력한다.
OncePerRequestFilter에서 헤더의 correlation ID를 검증하거나 새로 만들고 MDC에 넣는다.
필터 종료 시 finally 블록에서 MDC를 정리해 스레드 재사용에 따른 오염을 막는다.
예외는 stack trace와 안정적인 errorCode를 남기되 요청 본문 전체를 출력하지 않는다.
비밀번호, 세션 쿠키, Authorization 헤더, 카드 번호, 주민번호는 기록하지 않는다.

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

- 개발·스테이징·운영 Log Group을 분리하고 최소 권한을 적용한다.
- 운영 로그 보존 기간은 장애 조사, 법적 의무, 비용을 함께 고려한다.
- Logs Insights 저장 쿼리로 주문 실패와 외부 API 지연 조사를 표준화한다.
- 대량 검색 전에 시간 범위와 Log Group을 좁혀 스캔량을 줄인다.
- Subscription 대상 장애가 애플리케이션 요청을 막지 않도록 비동기로 구성한다.

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

무제한 보존으로 오래된 디버그 로그가 계속 쌓여 비용이 증가했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 2

DEBUG 로그를 운영에 상시 활성화해 처리량과 검색 노이즈가 급증했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 3

멀티라인 stack trace 파싱이 깨져 같은 예외가 여러 사건처럼 보였다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 4

민감정보가 로그에 포함되어 접근 통제와 삭제 대응이 필요해졌다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

---

# 비용과 성능 고려사항

- 수집 데이터 양, 보관량, Insights 스캔량, 구독 전달과 내보내기가 과금 요소이다.
- 로그 레벨과 샘플링을 조정하되 오류 원인과 감사 정보는 보존한다.
- 반복되는 정상 로그를 메트릭으로 대체하고 로그 메시지 크기를 제한한다.
- 압축과 장기 보관 목적이면 S3 전달 및 수명 주기를 검토한다.

정확한 단가는 Region과 시점에 따라 달라지므로 공식 요금 페이지와 실제 사용량으로 확인한다.

비용 최적화는 데이터를 무조건 버리는 일이 아니라 질문에 답하지 못하는 데이터를 줄이는 일이다.

---

# 기억할 내용

- Log Group과 Log Stream의 관계를 설명할 수 있다.
- JSON 구조화 로그와 보존 기간을 설계할 수 있다.
- Logs Insights로 장애 문맥을 검색할 수 있다.
- Subscription Filter와 Metric Filter의 차이를 구분할 수 있다.
- 로그·메트릭·트레이스는 공통 서비스와 환경 정보로 연결한다.
- 개인정보와 인증 정보는 관측 데이터에 남기지 않는다.
- 수집량과 카디널리티와 보존 기간을 운영 정책으로 관리한다.
- 알람은 소유자와 runbook이 있을 때 행동 가능한 신호가 된다.

---

# 다음 Chapter

다음 Chapter에서는 [CloudWatch Metrics](/aws-backend/part-11/03-cloudwatch-metrics/)를 학습한다.

현재 Chapter의 신호를 다음 단계의 운영 판단과 연결한다.
