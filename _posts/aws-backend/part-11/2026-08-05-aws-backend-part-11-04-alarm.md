---
title: "Chapter 04. CloudWatch Alarm"
permalink: /aws-backend/part-11/04-alarm/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. CloudWatch Alarm
## 노이즈를 줄이고 행동 가능한 장애 신호를 만드는 방법

> **학습 목표**
>
> - M out of N 평가 방식을 설명할 수 있다.
> - Missing Data 처리 정책을 선택할 수 있다.
> - Composite Alarm과 SNS 알림을 구성할 수 있다.
> - 알람과 오토스케일링의 목적을 구분할 수 있다.

---

# 왜 필요한가

행사 중 결제 실패율이 순간적으로 튀었다가 회복한다고 가정한다.
한 구간만으로 호출하면 담당자는 정상 변동에도 반복해서 깨게 된다.
최근 5개 구간 중 3개가 임계값을 넘을 때만 알리면 지속 장애에 집중할 수 있다.
알람이 울리면 영향, 원인 후보, 대시보드와 runbook이 함께 전달되어야 한다.
경보 수보다 사람이 즉시 행동할 수 있는지가 중요하다.

관측 데이터는 많이 모으는 것이 목적이 아니라 고객 영향과 다음 행동을 빠르게 결정하기 위한 근거이다.

---

# CloudWatch Alarm이란?

CloudWatch Alarm은 메트릭 또는 표현식의 평가 결과를 OK, ALARM, INSUFFICIENT_DATA 상태로 관리하고 상태 변경에 작업을 연결한다.

신호의 이름과 책임 경계를 먼저 정해야 여러 팀이 같은 데이터를 같은 의미로 해석할 수 있다.

---

# 동작 흐름

```
Metric: PaymentFailureRate
       │  period 1m
       ▼
Alarm: 3 of 5 > 5%
  ├─ OK
  ├─ ALARM ──▶ SNS ──▶ On-call
  └─ INSUFFICIENT_DATA
       │
Composite Alarm
  주문 오류 AND 결제 오류
       └─ runbook / dashboard
```

흐름은 다음 순서로 이해한다.

1. 행사 중 결제 실패율이 순간적으로 튀었다가 회복한다고 가정한다.
2. 한 구간만으로 호출하면 담당자는 정상 변동에도 반복해서 깨게 된다.
3. 최근 5개 구간 중 3개가 임계값을 넘을 때만 알리면 지속 장애에 집중할 수 있다.
4. 알람이 울리면 영향, 원인 후보, 대시보드와 runbook이 함께 전달되어야 한다.

각 단계는 실패할 수 있으므로 수집 누락과 전달 지연도 별도 상태로 다룬다.

---

# 핵심 개념

| 개념 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| Evaluation Periods | 평가할 전체 구간 N | 5개 |
| Datapoints to Alarm | 위반해야 할 구간 M | 3개 |
| Missing Data | 누락값 해석 | missing, breaching 등 |
| Composite Alarm | 여러 알람의 논리 조합 | AND, OR, NOT |
| SNS | 상태 변경 전달 | 온콜 시스템 연결 |
| Alarm Action | 알림 또는 자동 작업 | SNS, EC2 action |

개념을 독립적으로 외우기보다 하나의 장애 질문에 어떤 개념이 답하는지 연결한다.

---

# AWS CLI 설정과 조회

다음 명령은 이름과 ARN을 실습 환경에 맞게 바꿔 실행한다.

```bash
aws cloudwatch put-metric-alarm --alarm-name prod-order-high-errors --namespace ShoppingMall/Order --metric-name ErrorRate --statistic Average --period 60 --evaluation-periods 5 --datapoints-to-alarm 3 --threshold 5 --comparison-operator GreaterThanThreshold --treat-missing-data missing --alarm-actions arn:aws:sns:ap-northeast-2:123456789012:ops-alert
aws cloudwatch describe-alarms --alarm-names prod-order-high-errors
```

CLI 실행 역할에는 조회와 변경에 필요한 최소 권한만 부여한다.

운영 변경은 콘솔에서 임의로 수행하지 않고 IaC와 리뷰 절차로 재현 가능하게 관리한다.

---

# Spring Boot 3.x와 Java 17 연동

애플리케이션은 안정적인 errorCode와 Counter를 발행하고 알람 평가는 CloudWatch에 맡긴다.
Health Indicator는 의존성 상태를 제공하지만 매 요청마다 무거운 검사를 수행하지 않는다.
생성자 주입으로 MeterRegistry를 받아 결제 결과 메트릭을 기록한다.
알람 메시지에 배포 버전과 서비스 태그를 연결할 수 있도록 공통 tag를 사용한다.
코드 내부에서 직접 SNS를 호출해 모니터링 알림을 만들지 않는다.

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

- 페이지 알람은 즉시 고객 영향과 사람의 조치가 필요한 상태로 제한한다.
- 티켓 알람은 추세 악화처럼 근무 시간에 처리할 수 있는 상태로 분리한다.
- 모든 알람에 소유자, 심각도, 대시보드, runbook, 검증 방법을 기록한다.
- 정기적으로 무응답 알람과 반복 오탐을 폐기하거나 개선한다.
- 배포와 유지보수 중에는 억제 규칙을 사용하되 복구 시 반드시 해제한다.

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

트래픽이 없는 시간을 Missing Data=breaching으로 처리해 매일 야간 경보가 발생했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 2

모든 인스턴스 CPU 알람이 동시에 울려 하나의 장애가 수십 건으로 증폭되었다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 3

SNS 구독 확인이 끝나지 않아 실제 담당자에게 알림이 전달되지 않았다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 4

알람을 오토스케일링 정책과 동일시해 확장 후에도 고객 오류를 놓쳤다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

---

# 비용과 성능 고려사항

- 표준·고해상도 알람 수, 복합 알람, 메트릭과 알림 전달이 비용에 영향을 준다.
- 중복 인스턴스 알람을 서비스 수준 Composite Alarm으로 묶어 노이즈를 낮춘다.
- 짧은 Period는 빠르게 감지하지만 변동성과 평가 비용을 높일 수 있다.
- 알람 비용 절감보다 탐지 지연과 온콜 피로의 균형을 우선한다.

정확한 단가는 Region과 시점에 따라 달라지므로 공식 요금 페이지와 실제 사용량으로 확인한다.

비용 최적화는 데이터를 무조건 버리는 일이 아니라 질문에 답하지 못하는 데이터를 줄이는 일이다.

---

# 기억할 내용

- M out of N 평가 방식을 설명할 수 있다.
- Missing Data 처리 정책을 선택할 수 있다.
- Composite Alarm과 SNS 알림을 구성할 수 있다.
- 알람과 오토스케일링의 목적을 구분할 수 있다.
- 로그·메트릭·트레이스는 공통 서비스와 환경 정보로 연결한다.
- 개인정보와 인증 정보는 관측 데이터에 남기지 않는다.
- 수집량과 카디널리티와 보존 기간을 운영 정책으로 관리한다.
- 알람은 소유자와 runbook이 있을 때 행동 가능한 신호가 된다.

---

# 다음 Chapter

다음 Chapter에서는 [CloudWatch Dashboard](/aws-backend/part-11/05-dashboard/)를 학습한다.

현재 Chapter의 신호를 다음 단계의 운영 판단과 연결한다.
