---
title: "Chapter 08. Security Operations"
permalink: /aws-backend/part-12/08-security-operations/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Security Operations
## 탐지에서 복구까지 이어지는 운영

> **학습 목표**
>
> - 주요 보안 서비스의 역할을 구분할 수 있다.
> - 탐지 결과를 증거와 영향 범위로 연결할 수 있다.
> - Incident Response 흐름을 설계할 수 있다.
> - 자동 대응의 권한과 실패 위험을 설명할 수 있다.

---

# 침해 시나리오

평소 사용하지 않던 Region에서 IAM Access Key로 대량 API 호출이 발생했다고 가정해 보자.

CloudTrail은 누가 어떤 API를 호출했는지 기록하고 GuardDuty는 행위와 위협 정보를 분석해 Finding을 만들 수 있다.

Security Hub는 여러 Finding과 보안 기준 결과를 모아 우선순위를 관리한다.

담당자는 키를 차단하는 동시에 증거를 보존하고 영향 Resource와 데이터 접근 범위를 조사해야 한다.

---

# 서비스 역할 비교

| 서비스 | 핵심 역할 | 대표 질문 |
|---|---|---|
| CloudTrail | API 활동 기록 | 누가 언제 무엇을 호출했는가 |
| Config | Resource 구성과 변경 평가 | 구성이 언제 어떻게 바뀌었는가 |
| IAM Access Analyzer | 외부 접근과 미사용 권한 분석 | 의도하지 않은 공유가 있는가 |
| GuardDuty | 위협 신호와 행위 탐지 | 침해로 의심할 활동이 있는가 |
| Security Hub | Finding 집계와 보안 상태 관리 | 무엇부터 대응해야 하는가 |

이 서비스들은 서로 대체하지 않으며 기록, 구성, 권한, 위협, 통합이라는 다른 관점을 제공한다.

---

# 운영 구조

```text
AWS Accounts
  ├── CloudTrail ───────▶ 중앙 Log Archive
  ├── Config ───────────▶ 구성 이력과 Rule
  ├── Access Analyzer ──▶ 외부 접근 Finding
  └── GuardDuty ────────▶ 위협 Finding
                              │
                              ▼
                         Security Hub
                              │
                              ▼
                     EventBridge / Triage
                              │
                              ▼
                   격리 ─ 조사 ─ 복구 ─ 회고
```

중앙 보안 계정은 로그와 Finding을 Workload 계정의 침해로부터 분리한다.

---

# CloudTrail

CloudTrail Management Event는 Resource 생성, 정책 변경 같은 제어 영역 활동을 기록한다.

Data Event는 S3 Object, Lambda 호출 같은 데이터 영역 활동을 더 세밀하게 기록할 수 있다.

Organization Trail과 중앙 저장소를 사용하고 로그 삭제와 Trail 중지에 경보를 건다.

로그에는 요청 시간, Source IP, User Agent, Identity, Request Parameter와 오류가 포함될 수 있다.

민감 값이 요청 파라미터에 들어가지 않도록 애플리케이션과 운영 명령을 설계한다.

---

# IAM Access Analyzer

Access Analyzer는 Resource Policy를 분석해 조직이나 계정 밖에서 접근 가능한 Resource를 찾는다.

S3 Bucket, IAM Role, KMS Key 등 지원 Resource의 외부 공유가 의도된 것인지 검토한다.

Policy Validation으로 오류와 보안 권고를 배포 전에 확인한다.

사용 기록 기반 정책 생성과 미사용 접근 분석은 최소 권한 축소의 근거가 된다.

Finding이 있다는 사실만으로 공개 침해를 의미하지는 않으며 승인된 공유인지 판단해야 한다.

---

# GuardDuty

GuardDuty는 CloudTrail 활동, 네트워크와 DNS 등 지원되는 데이터 소스를 분석해 위협 Finding을 생성한다.

Finding은 탐지 신호이며 최종 침해 판정이 아니므로 원본 증거와 환경 문맥으로 검증한다.

심각도, 영향 Resource, 최초와 마지막 관측 시간, 관련 Principal을 Triage에 사용한다.

억제 규칙은 알려진 정상 패턴에 제한하고 실제 공격을 숨기지 않는지 정기 검토한다.

---

# Security Hub와 Config

Security Hub는 여러 계정과 Region의 Finding과 보안 기준 점검을 집계한다.

통합 화면만 켜는 것으로 대응이 끝나지 않으며 소유자, SLA, 상태 전환과 Ticket 연결이 필요하다.

Config는 Resource 구성 변경 이력과 Rule 준수 상태를 제공해 공개 설정이나 암호화 누락을 찾는다.

Config의 비준수는 취약한 구성 신호이고 GuardDuty Finding은 위협 행위 신호라는 차이가 있다.

---

# Incident Response 흐름

```text
준비
  ▼
탐지와 신고
  ▼
Triage와 심각도 결정
  ▼
격리와 자격 증명 차단
  ▼
증거 보존과 영향 범위 조사
  ▼
원인 제거와 안전한 복구
  ▼
모니터링 강화와 회고
```

격리는 공격 확산을 막되 증거와 정상 서비스 가용성을 불필요하게 파괴하지 않아야 한다.

복구는 깨끗한 이미지와 검증된 설정으로 수행하고 침해된 서버를 그대로 재사용하지 않는다.

---

# CLI 조사 예시

```bash
aws sts get-caller-identity

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAEXAMPLE

aws guardduty list-findings \
  --detector-id DETECTOR_ID

aws accessanalyzer list-findings-v2 \
  --analyzer-arn ANALYZER_ARN
```

실제 조사에서는 시간대를 통일하고 명령, 결과 위치, 담당자와 판단 근거를 사건 Timeline에 남긴다.

---

# 자동 대응

EventBridge는 Finding을 Lambda나 Step Functions 대응 흐름으로 전달할 수 있다.

Access Key 비활성화, Security Group 격리 같은 자동화는 빠르지만 오탐이면 정상 서비스를 중단시킨다.

높은 확신의 반복 작업만 자동화하고 위험한 조치는 사람 승인과 Rollback을 둔다.

대응 Role 자체가 공격 경로가 되지 않도록 대상 Resource와 허용 Action을 제한한다.

---

# Spring Boot의 역할

애플리케이션은 인증 실패, 권한 거부와 민감 작업을 구조화된 감사 이벤트로 남긴다.

비밀번호, Token, Secret 원문과 전체 요청 Body는 로그에 남기지 않는다.

```java
@Component
class SecurityAuditLogger {
    private static final Logger log =
            LoggerFactory.getLogger(SecurityAuditLogger.class);

    void accessDenied(String principal, String action) {
        log.warn("security_event=access_denied principal={} action={}",
                principal, action);
    }
}
```

감사 Logger는 생성자 주입 가능한 별도 Component로 두고 민감 필드 필터를 중앙 적용한다.

---

# 장애와 보안 사고

CloudTrail이 비어 있으면 Region, Event 유형, 조회 시간, Trail 상태와 권한을 확인한다.

Finding 폭증 시 무작정 모두 닫지 말고 공통 원인과 공격 Campaign을 묶어 우선순위를 정한다.

자동 격리 Lambda 실패에 대비해 수동 Runbook과 비상 Role을 준비한다.

조사 중 Root나 공용 관리자 계정을 사용하면 추가 행위와 공격자 행위를 구분하기 어렵다.

---

# 비용 고려사항

CloudTrail Data Event, Config 기록, GuardDuty 보호 기능, Security Hub 점검, 로그 저장과 조회가 비용에 영향을 준다.

정확한 가격과 지원 범위는 Region과 시점에 따라 달라지므로 공식 가격 페이지와 실제 사용량으로 계산한다.

위험 기반으로 데이터 소스와 보존 기간을 정하되 핵심 증거를 비용 때문에 누락하지 않는다.

---

# 기억해야 할 내용

- CloudTrail은 API 활동을 기록한다.
- Config는 Resource 구성 이력과 준수를 평가한다.
- Access Analyzer는 외부 접근과 미사용 권한을 분석한다.
- GuardDuty는 위협 신호를 탐지한다.
- Security Hub는 Finding과 보안 상태를 집계한다.
- 대응은 탐지, 격리, 조사, 제거, 복구와 회고로 이어진다.
- 자동 대응에는 최소 권한, 승인과 Rollback이 필요하다.

---

# 다음 Chapter

다음 Chapter에서는 Part 12의 권한, 암호화, 비밀 관리와 보안 운영을 하나의 흐름으로 정리한다.
