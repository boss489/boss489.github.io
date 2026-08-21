---
title: "Chapter 05. CloudWatch Dashboard"
permalink: /aws-backend/part-11/05-dashboard/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. CloudWatch Dashboard
## 기술 상태와 사업 영향을 한 화면에서 읽는 방법

> **학습 목표**
>
> - RED와 USE 방법을 구분할 수 있다.
> - 서비스 지표와 비즈니스 지표를 함께 배치할 수 있다.
> - 운영 역할별 대시보드를 설계할 수 있다.
> - 위젯에서 runbook과 상세 조사 화면으로 이동하게 만들 수 있다.

---

# 왜 필요한가

주문 전환율이 급락했지만 서버 CPU는 정상이라고 가정한다.
인프라 지표만 있는 화면은 고객이 결제를 끝내지 못하는 사실을 설명하지 못한다.
주문 성공률, 결제사별 실패율, p99 지연을 함께 보면 영향 범위를 좁힐 수 있다.
대시보드는 예쁜 차트 모음이 아니라 다음 행동으로 안내하는 운영 인터페이스이다.
첫 화면은 상태를 판단하고 세부 화면은 원인을 조사하도록 계층화한다.

관측 데이터는 많이 모으는 것이 목적이 아니라 고객 영향과 다음 행동을 빠르게 결정하기 위한 근거이다.

---

# CloudWatch Dashboard이란?

CloudWatch Dashboard는 여러 Region과 서비스의 메트릭, 로그 쿼리, 알람 상태, 텍스트를 위젯으로 구성하는 공유 가능한 운영 화면이다.

신호의 이름과 책임 경계를 먼저 정해야 여러 팀이 같은 데이터를 같은 의미로 해석할 수 있다.

---

# 동작 흐름

```
Executive: 주문량 / 매출 / 전환율
                │ drill down
                ▼
Service RED: Rate / Errors / Duration
                │
                ▼
Resource USE: Utilization / Saturation / Errors
                │
                ▼
Logs Insights / Trace / Runbook
```

흐름은 다음 순서로 이해한다.

1. 주문 전환율이 급락했지만 서버 CPU는 정상이라고 가정한다.
2. 인프라 지표만 있는 화면은 고객이 결제를 끝내지 못하는 사실을 설명하지 못한다.
3. 주문 성공률, 결제사별 실패율, p99 지연을 함께 보면 영향 범위를 좁힐 수 있다.
4. 대시보드는 예쁜 차트 모음이 아니라 다음 행동으로 안내하는 운영 인터페이스이다.

각 단계는 실패할 수 있으므로 수집 누락과 전달 지연도 별도 상태로 다룬다.

---

# 핵심 개념

| 개념 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| RED | Rate, Errors, Duration | 요청 기반 서비스 |
| USE | Utilization, Saturation, Errors | CPU·Pool·Queue 자원 |
| 서비스 지표 | API 품질 | 오류율과 p99 |
| 비즈니스 지표 | 고객 결과 | 주문 완료와 결제 성공률 |
| Text Widget | 설명과 링크 | 소유자와 runbook |
| 변수 | 화면 재사용 | 환경·서비스 선택 |

개념을 독립적으로 외우기보다 하나의 장애 질문에 어떤 개념이 답하는지 연결한다.

---

# AWS CLI 설정과 조회

다음 명령은 이름과 ARN을 실습 환경에 맞게 바꿔 실행한다.

```bash
aws cloudwatch put-dashboard --dashboard-name prod-shopping-overview --dashboard-body file://dashboard.json
aws cloudwatch get-dashboard --dashboard-name prod-shopping-overview
aws cloudwatch list-dashboards --dashboard-name-prefix prod-
```

CLI 실행 역할에는 조회와 변경에 필요한 최소 권한만 부여한다.

운영 변경은 콘솔에서 임의로 수행하지 않고 IaC와 리뷰 절차로 재현 가능하게 관리한다.

---

# Spring Boot 3.x와 Java 17 연동

Micrometer로 HTTP RED 지표와 커넥션 풀 USE 지표를 제공한다.
주문 완료와 결제 실패 같은 비즈니스 Counter를 도메인 서비스에서 명시적으로 기록한다.
MeterRegistry는 생성자 주입하고 계층별 공통 tag는 설정 클래스에서 관리한다.
대시보드용 메트릭 때문에 사용자 식별자를 tag로 추가하지 않는다.
배포 버전 정보는 Actuator info와 공통 메트릭 tag로 연결한다.

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

- 최상단에는 현재 고객 영향과 핵심 알람 상태를 배치한다.
- 모든 그래프에 단위, 기간, Statistic, timezone을 명확히 표시한다.
- Text 위젯에 소유 팀, 최근 배포, runbook, 로그 쿼리 링크를 둔다.
- 같은 시간축으로 그래프를 정렬해 원인과 결과의 선후를 비교한다.
- 대시보드를 코드로 관리해 리뷰와 환경 간 재현성을 확보한다.

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

서로 다른 Period의 그래프를 비교해 장애 시점을 잘못 판단했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 2

누적 주문 수를 그대로 그려 최근 감소를 알아보기 어려웠다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 3

위젯이 너무 많아 핵심 고객 영향이 화면 아래에 묻혔다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 4

수동 편집만 사용해 운영과 스테이징 화면의 정의가 달라졌다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

---

# 비용과 성능 고려사항

- 대시보드 수, 위젯의 메트릭 조회, 로그 쿼리와 교차 계정 관측이 비용에 영향을 준다.
- 항상 필요한 화면과 장애 때만 여는 상세 화면을 분리한다.
- 불필요한 자동 새로고침과 과도한 고해상도 조회를 피한다.
- 중복 대시보드를 템플릿과 변수로 통합하되 소유권은 명확히 유지한다.

정확한 단가는 Region과 시점에 따라 달라지므로 공식 요금 페이지와 실제 사용량으로 확인한다.

비용 최적화는 데이터를 무조건 버리는 일이 아니라 질문에 답하지 못하는 데이터를 줄이는 일이다.

---

# 기억할 내용

- RED와 USE 방법을 구분할 수 있다.
- 서비스 지표와 비즈니스 지표를 함께 배치할 수 있다.
- 운영 역할별 대시보드를 설계할 수 있다.
- 위젯에서 runbook과 상세 조사 화면으로 이동하게 만들 수 있다.
- 로그·메트릭·트레이스는 공통 서비스와 환경 정보로 연결한다.
- 개인정보와 인증 정보는 관측 데이터에 남기지 않는다.
- 수집량과 카디널리티와 보존 기간을 운영 정책으로 관리한다.
- 알람은 소유자와 runbook이 있을 때 행동 가능한 신호가 된다.

---

# 다음 Chapter

다음 Chapter에서는 [CloudTrail](/aws-backend/part-11/06-cloudtrail/)를 학습한다.

현재 Chapter의 신호를 다음 단계의 운영 판단과 연결한다.
