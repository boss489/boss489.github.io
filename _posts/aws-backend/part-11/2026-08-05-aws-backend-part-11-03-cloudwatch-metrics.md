---
title: "Chapter 03. CloudWatch Metrics"
permalink: /aws-backend/part-11/03-cloudwatch-metrics/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. CloudWatch Metrics
## 시간에 따른 서비스 상태를 숫자로 표현하는 방법

> **학습 목표**
>
> - Namespace, Dimension, Statistic, Period를 설명할 수 있다.
> - 기본 메트릭과 Custom Metric을 구분할 수 있다.
> - Embedded Metric Format을 적용할 수 있다.
> - 평균과 percentile을 올바르게 해석할 수 있다.

---

# 왜 필요한가

주문 API가 느리다는 문의가 간헐적으로 발생한다고 가정한다.
평균 응답 시간만 보면 빠른 요청이 느린 요청을 가려 정상처럼 보일 수 있다.
p95와 p99를 보면 꼬리 지연으로 영향을 받는 고객 비율을 판단할 수 있다.
결제 실패율과 주문 완료 수를 함께 보면 기술 상태와 사업 영향을 연결할 수 있다.
메트릭은 질문에 답할 수 있는 이름과 단위와 차원을 가져야 한다.

관측 데이터는 많이 모으는 것이 목적이 아니라 고객 영향과 다음 행동을 빠르게 결정하기 위한 근거이다.

---

# CloudWatch Metrics이란?

CloudWatch Metric은 Namespace 안에서 이름, Dimension, timestamp, value, unit으로 식별되는 시계열 데이터이다.

신호의 이름과 책임 경계를 먼저 정해야 여러 팀이 같은 데이터를 같은 의미로 해석할 수 있다.

---

# 동작 흐름

```
Spring Boot / ECS / ALB
       │ datapoints
       ▼
Namespace ShoppingMall/Order
  ├─ Metric: CheckoutLatency
  ├─ Dimensions: Service=order, Env=prod
  ├─ Period: 60 seconds
  └─ Statistics: Sum, Average, p95, p99
       │
       ├─ Dashboard
       └─ Alarm
```

흐름은 다음 순서로 이해한다.

1. 주문 API가 느리다는 문의가 간헐적으로 발생한다고 가정한다.
2. 평균 응답 시간만 보면 빠른 요청이 느린 요청을 가려 정상처럼 보일 수 있다.
3. p95와 p99를 보면 꼬리 지연으로 영향을 받는 고객 비율을 판단할 수 있다.
4. 결제 실패율과 주문 완료 수를 함께 보면 기술 상태와 사업 영향을 연결할 수 있다.

각 단계는 실패할 수 있으므로 수집 누락과 전달 지연도 별도 상태로 다룬다.

---

# 핵심 개념

| 개념 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| Namespace | 메트릭 논리 그룹 | ShoppingMall/Order |
| Dimension | 시계열을 나누는 키·값 | Service=order |
| Statistic | 기간 내 집계 방식 | Sum, Average, p99 |
| Period | 집계 시간 창 | 60초 |
| 기본 메트릭 | AWS 서비스가 제공 | ALB RequestCount |
| Custom Metric | 업무에 맞춰 발행 | PaymentFailure |

개념을 독립적으로 외우기보다 하나의 장애 질문에 어떤 개념이 답하는지 연결한다.

---

# AWS CLI 설정과 조회

다음 명령은 이름과 ARN을 실습 환경에 맞게 바꿔 실행한다.

```bash
aws cloudwatch put-metric-data --namespace ShoppingMall/Order --metric-data 'MetricName=CheckoutSuccess,Dimensions=[{Name=Environment,Value=prod}],Value=1,Unit=Count'
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name TargetResponseTime --dimensions Name=LoadBalancer,Value=app/shop/abc --start-time 2026-08-05T00:00:00Z --end-time 2026-08-05T01:00:00Z --period 300 --extended-statistics p99
```

CLI 실행 역할에는 조회와 변경에 필요한 최소 권한만 부여한다.

운영 변경은 콘솔에서 임의로 수행하지 않고 IaC와 리뷰 절차로 재현 가능하게 관리한다.

---

# Spring Boot 3.x와 Java 17 연동

Micrometer Counter로 주문 성공과 실패를 단조 증가 값으로 기록한다.
Timer로 API와 결제 호출 지연 분포를 측정하고 필요한 percentile을 구성한다.
CloudWatch MeterRegistry는 단계별 집계를 CloudWatch Custom Metric으로 전송한다.
EMF는 JSON 로그 안에 메트릭 선언을 넣어 로그와 메트릭을 함께 수집하는 방식이다.
orderId와 userId 같은 무한한 값을 tag 또는 Dimension으로 사용하지 않는다.

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

- Count는 Sum, 사용률은 Average, 지연은 percentile을 우선 검토한다.
- Period가 너무 길면 짧은 장애가 평균에 묻히고 너무 짧으면 변동성과 비용이 커진다.
- 메트릭 이름, 단위, tag 허용 목록을 플랫폼 표준으로 관리한다.
- 배포 전후 동일한 기간과 트래픽 조건을 비교한다.
- 0과 데이터 없음은 의미가 다르므로 발행 방식과 알람 정책을 맞춘다.

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

주문 ID를 Dimension으로 발행해 시계열 수와 비용이 폭증했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 2

오류율 대신 오류 건수만 보아 야간의 적은 오류를 과대평가했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 3

Average만 사용해 p99 결제 지연을 놓쳤다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 4

단위가 Milliseconds와 Seconds로 섞여 잘못된 임계값을 설정했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

---

# 비용과 성능 고려사항

- Custom Metric 수, API 호출, 고해상도 데이터, 조회와 스트림이 과금 요소이다.
- Dimension 조합마다 별도 시계열이 되므로 카디널리티 예산을 둔다.
- EMF는 로그 수집과 저장 비용도 함께 발생하므로 중복 로그를 통제한다.
- 필요한 해상도와 보존 목적을 기준으로 Period와 집계를 정한다.

정확한 단가는 Region과 시점에 따라 달라지므로 공식 요금 페이지와 실제 사용량으로 확인한다.

비용 최적화는 데이터를 무조건 버리는 일이 아니라 질문에 답하지 못하는 데이터를 줄이는 일이다.

---

# 기억할 내용

- Namespace, Dimension, Statistic, Period를 설명할 수 있다.
- 기본 메트릭과 Custom Metric을 구분할 수 있다.
- Embedded Metric Format을 적용할 수 있다.
- 평균과 percentile을 올바르게 해석할 수 있다.
- 로그·메트릭·트레이스는 공통 서비스와 환경 정보로 연결한다.
- 개인정보와 인증 정보는 관측 데이터에 남기지 않는다.
- 수집량과 카디널리티와 보존 기간을 운영 정책으로 관리한다.
- 알람은 소유자와 runbook이 있을 때 행동 가능한 신호가 된다.

---

# 다음 Chapter

다음 Chapter에서는 [CloudWatch Alarm](/aws-backend/part-11/04-alarm/)를 학습한다.

현재 Chapter의 신호를 다음 단계의 운영 판단과 연결한다.
