---
title: "Chapter 06. CloudTrail"
permalink: /aws-backend/part-11/06-cloudtrail/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. CloudTrail
## 누가 AWS 리소스를 어떻게 변경했는지 추적하는 방법

> **학습 목표**
>
> - Management Event와 Data Event를 구분할 수 있다.
> - Event History와 Trail의 차이를 설명할 수 있다.
> - 조직 Trail과 로그 무결성 검증을 설계할 수 있다.
> - S3와 CloudWatch Logs 전달 목적을 구분할 수 있다.

---

# 왜 필요한가

운영 보안 그룹의 인바운드 규칙이 갑자기 열렸다고 가정한다.
애플리케이션 로그는 AWS 제어 영역의 변경 주체와 호출 API를 알려주지 않는다.
CloudTrail에서 principal, source IP, eventName, requestParameters를 확인해야 한다.
여러 계정을 각각 조사하면 누락이 생기므로 조직 단위 Trail로 중앙 수집한다.
감사 로그는 운영 계정과 분리된 저장소에서 삭제 방지와 무결성을 확보해야 한다.

관측 데이터는 많이 모으는 것이 목적이 아니라 고객 영향과 다음 행동을 빠르게 결정하기 위한 근거이다.

---

# CloudTrail이란?

AWS CloudTrail은 AWS 계정의 API 호출과 콘솔 활동을 이벤트로 기록해 거버넌스, 감사, 보안 조사에 사용하는 서비스이다.

신호의 이름과 책임 경계를 먼저 정해야 여러 팀이 같은 데이터를 같은 의미로 해석할 수 있다.

---

# 동작 흐름

```
User / Role / AWS Service
          │ API call
          ▼
      AWS Control Plane
          │
          ▼
       CloudTrail
        ├─ Event History
        └─ Organization Trail
             ├─ S3 immutable archive
             └─ CloudWatch Logs ──▶ Alarm
```

흐름은 다음 순서로 이해한다.

1. 운영 보안 그룹의 인바운드 규칙이 갑자기 열렸다고 가정한다.
2. 애플리케이션 로그는 AWS 제어 영역의 변경 주체와 호출 API를 알려주지 않는다.
3. CloudTrail에서 principal, source IP, eventName, requestParameters를 확인해야 한다.
4. 여러 계정을 각각 조사하면 누락이 생기므로 조직 단위 Trail로 중앙 수집한다.

각 단계는 실패할 수 있으므로 수집 누락과 전달 지연도 별도 상태로 다룬다.

---

# 핵심 개념

| 개념 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| Management Event | 리소스 구성 작업 | CreateRole, RunInstances |
| Data Event | 리소스 내부 데이터 작업 | S3 GetObject, Lambda Invoke |
| Event History | 최근 관리 이벤트 조회 | 빠른 조사 |
| Trail | 지속 수집과 전달 | S3 또는 Logs |
| Organization Trail | 조직 계정 중앙 수집 | 보안 계정 |
| Log Validation | digest로 무결성 확인 | 변조 탐지 |

개념을 독립적으로 외우기보다 하나의 장애 질문에 어떤 개념이 답하는지 연결한다.

---

# AWS CLI 설정과 조회

다음 명령은 이름과 ARN을 실습 환경에 맞게 바꿔 실행한다.

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress
aws cloudtrail create-trail --name organization-audit --s3-bucket-name central-audit-logs --is-organization-trail --enable-log-file-validation
aws cloudtrail start-logging --name organization-audit
aws cloudtrail validate-logs --trail-arn arn:aws:cloudtrail:ap-northeast-2:123456789012:trail/organization-audit --start-time 2026-08-05T00:00:00Z --end-time 2026-08-06T00:00:00Z
```

CLI 실행 역할에는 조회와 변경에 필요한 최소 권한만 부여한다.

운영 변경은 콘솔에서 임의로 수행하지 않고 IaC와 리뷰 절차로 재현 가능하게 관리한다.

---

# Spring Boot 3.x와 Java 17 연동

Spring Boot는 CloudTrail 이벤트를 직접 생성하지 않으며 AWS SDK 호출이 AWS 측에서 기록된다.
애플리케이션 감사 로그에는 주문 상태를 바꾼 업무 주체와 이유를 별도로 남긴다.
CloudTrail의 IAM principal과 업무 사용자 ID는 서로 다른 계층의 감사 정보이다.
AWS SDK client는 생성자 주입하고 역할 기반 임시 자격 증명을 사용한다.
애플리케이션 로그의 requestId를 AWS SDK 호출 문맥과 연결하면 조사가 쉬워진다.

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

- 모든 Region을 포함하는 조직 Trail을 보안 전용 계정으로 전달한다.
- S3 Bucket Policy와 KMS Key Policy에서 CloudTrail 쓰기와 감사 읽기를 분리한다.
- Data Event는 중요한 Bucket과 함수부터 선택해 수집 범위를 통제한다.
- Trail 중지, 삭제, 정책 변경 자체를 즉시 경보한다.
- CloudWatch Logs는 빠른 탐지에, S3는 장기 감사와 분석에 사용한다.

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

Event History만 믿어 장기 감사 시점의 기록을 찾지 못했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 2

S3 Data Event를 전부 켜 대량 객체 접근으로 이벤트와 비용이 급증했다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 3

Trail과 로그 Bucket이 같은 운영 권한에 있어 공격자가 함께 삭제할 수 있었다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

## 사례 4

로그 파일 검증을 활성화했지만 실제 검증 절차를 runbook에 넣지 않았다.

원인과 증상을 분리하고 재현 가능한 데이터로 판단해야 한다.

대응 후 동일 조건의 회귀 알람과 runbook을 보완한다.

---

# 비용과 성능 고려사항

- Trail 유형, Management Event의 추가 복사본, Data Event 양, Insights, S3와 Logs 저장이 과금 요소이다.
- 대량 Data Event는 리소스와 작업 유형을 선별한다.
- 장기 보관은 S3 Lifecycle과 보존 정책으로 관리한다.
- 보안 증거의 완전성을 훼손하는 비용 절감은 허용하지 않는다.

정확한 단가는 Region과 시점에 따라 달라지므로 공식 요금 페이지와 실제 사용량으로 확인한다.

비용 최적화는 데이터를 무조건 버리는 일이 아니라 질문에 답하지 못하는 데이터를 줄이는 일이다.

---

# 기억할 내용

- Management Event와 Data Event를 구분할 수 있다.
- Event History와 Trail의 차이를 설명할 수 있다.
- 조직 Trail과 로그 무결성 검증을 설계할 수 있다.
- S3와 CloudWatch Logs 전달 목적을 구분할 수 있다.
- 로그·메트릭·트레이스는 공통 서비스와 환경 정보로 연결한다.
- 개인정보와 인증 정보는 관측 데이터에 남기지 않는다.
- 수집량과 카디널리티와 보존 기간을 운영 정책으로 관리한다.
- 알람은 소유자와 runbook이 있을 때 행동 가능한 신호가 된다.

---

# 다음 Chapter

다음 Chapter에서는 [X-Ray Distributed Tracing](/aws-backend/part-11/07-xray-distributed-tracing/)를 학습한다.

현재 Chapter의 신호를 다음 단계의 운영 판단과 연결한다.
