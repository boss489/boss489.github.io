---
title: "Chapter 02. IAM User와 Group"
permalink: /aws-backend/part-12/02-iam-user-group/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. IAM User와 Group
## 사람의 접근을 안전하게 관리하는 방법

> **학습 목표**
>
> - IAM User와 Group의 책임을 구분할 수 있다.
> - Root user와 관리자의 인증을 안전하게 구성할 수 있다.
> - 사람과 Workload에 서로 다른 자격 증명을 선택할 수 있다.
> - 퇴사자와 유출 키 사고에 대응할 수 있다.

---

# 권한 문제 시나리오

개발자마다 직접 정책을 붙이고 공용 액세스 키까지 공유하는 팀을 가정해 보자.

한 명이 퇴사해도 어떤 키를 폐기해야 하는지 알 수 없고 공유 키를 폐기하면 모든 자동화가 중단된다.

개별 정책이 누적되면 같은 직무의 사용자도 서로 다른 권한을 가져 검토와 감사가 어려워진다.

사람의 신원을 개별적으로 식별하고 직무 권한을 Group으로 관리해야 책임 추적과 회수가 가능하다.

---

# IAM User와 Group

IAM User는 AWS 계정 안에서 한 사람 또는 예외적인 장기 주체를 나타내는 Identity다.

IAM Group은 여러 User에 공통 권한을 적용하기 위한 논리적 묶음이며 스스로 로그인하거나 API를 호출하지 않는다.

| 기준 | User | Group | Role |
|---|---|---|---|
| 주 대상 | 사람 | 같은 직무의 사람 | Workload와 위임 주체 |
| 자격 증명 | 콘솔 암호와 액세스 키 가능 | 없음 | STS 임시 자격 증명 |
| 정책 연결 | 가능 | 가능 | 가능 |
| 멤버 관계 | Group에 소속 | User 포함 | AssumeRole로 전환 |
| 권장 사용 | 사람의 개별 식별 | 직무 권한 묶음 | 애플리케이션 실행 |

조직 규모가 커지면 IAM User를 직접 늘리기보다 IAM Identity Center와 외부 Identity Provider의 Federation을 우선 검토한다.

Workload에는 User를 만들지 않고 Role을 사용한다.

---

# 구조

```
Identity Provider
       │ SSO
       ▼
IAM Identity Center
       │ Permission Set
       ▼
AWS Account Role

예외적인 IAM User
       │
       ├──▶ Developers Group ── ReadOnly Policy
       └──▶ Operators Group  ── Operations Policy
```

Group은 직무를 표현하고 프로젝트마다 임시로 필요한 권한은 Role 전환으로 분리하는 편이 좋다.

---

# Root user 보호

Root user는 계정 생성 이메일과 암호로 로그인하며 모든 권한의 최종 소유자다.

Root에는 강력한 MFA를 등록하고 복구 수단과 이메일 계정을 별도로 보호한다.

Root 액세스 키는 생성하지 않고 이미 존재한다면 사용 여부를 조사한 뒤 제거한다.

Root만 가능한 제한된 계정 작업 외에는 관리 작업에도 Root를 사용하지 않는다.

Root 로그인과 MFA 변경 이벤트는 즉시 경보하도록 구성한다.

---

# User와 Group 생성

```bash
aws iam create-group --group-name Developers

aws iam create-user --user-name minsu

aws iam add-user-to-group \
  --group-name Developers \
  --user-name minsu

aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

AWS 관리형 정책은 시작점으로 편리하지만 운영에서는 실제 업무에 맞춘 고객 관리형 정책으로 줄여야 한다.

---

# MFA를 조건으로 요구하는 정책

다음 예시는 MFA가 없는 세션에서 제한된 자기 관리 작업 외의 API를 거부하는 핵심 조건을 보여준다.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyWithoutMfa",
    "Effect": "Deny",
    "NotAction": [
      "iam:CreateVirtualMFADevice",
      "iam:EnableMFADevice",
      "iam:GetUser",
      "sts:GetSessionToken"
    ],
    "Resource": "*",
    "Condition": {
      "BoolIfExists": {
        "aws:MultiFactorAuthPresent": "false"
      }
    }
  }]
}
```

실제 적용 전에는 비상 접근 절차와 정책 시뮬레이션으로 잠금 위험을 검증해야 한다.

---

# 액세스 키 관리

액세스 키는 콘솔 암호와 별개인 API 자격 증명이며 Secret Access Key는 생성 시점에만 확인할 수 있다.

사람도 가능하면 SSO와 단기 세션을 사용하고 장기 액세스 키 발급은 예외로 관리한다.

키에는 소유자, 목적, 만료 계획을 기록하고 미사용 키를 주기적으로 탐지한다.

키를 교체할 때는 새 키 배포, 정상 호출 확인, 이전 키 비활성화, 모니터링, 삭제 순서를 따른다.

Git 기록에 들어간 키는 현재 파일에서 지워도 유출된 것으로 간주하고 즉시 폐기한다.

---

# Spring Boot에서 피해야 할 구성

다음과 같이 장기 키를 설정 파일에 저장하면 Git, 빌드 산출물, 로그를 통해 유출될 수 있다.

```yaml
# 금지 예시
aws:
  access-key: AKIA...
  secret-key: plaintext-secret
```

애플리케이션은 실행 환경 Role과 `DefaultCredentialsProvider`를 사용해야 한다.

```java
@Configuration
class AwsClientConfiguration {
    @Bean
    DynamoDbClient dynamoDbClient() {
        return DynamoDbClient.builder()
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}
```

생성자 주입으로 SDK Client를 Service에 전달하면 자격 증명 로직이 비즈니스 코드에 퍼지지 않는다.

---

# 입사와 퇴사 운영

입사 시 개인 Identity를 만들고 직무 Group 또는 Permission Set만 부여한다.

권한 상승은 승인 기록, 유효 기간, 작업 사유를 남기는 Role 전환으로 처리한다.

부서 이동 시 이전 직무 권한을 먼저 제거하고 새 권한을 추가한다.

퇴사 시 SSO 세션, 콘솔 암호, 액세스 키, SSH Key, 애플리케이션 토큰을 한 절차에서 회수한다.

CloudTrail의 마지막 활동과 Access Advisor를 확인해 숨은 의존성을 찾는다.

---

# 장애와 보안 사고

Group 정책 변경 뒤 배포가 실패하면 사용자가 실제로 어느 Group과 Role을 통하는지부터 확인한다.

유출 키가 발견되면 비활성화 후 CloudTrail에서 Access Key ID로 호출 내역과 생성 자원을 조사한다.

관리자 MFA 장치를 분실하면 검증된 복구 절차를 사용하고 편의를 위해 MFA 요구 정책을 전체 해제하지 않는다.

공용 계정은 행위자를 식별할 수 없으므로 사고 조사와 책임 분리를 무너뜨린다.

---

# 비용 고려사항

IAM User와 Group 자체보다 MFA 장치 운영, 외부 Identity Provider, 감사 로그 저장과 검토에 비용이 든다.

관리 비용까지 고려하면 다수의 개별 User보다 중앙 Federation이 계정 수가 많은 조직에 유리하다.

저렴하다는 이유로 공유 계정을 사용하면 회수와 사고 조사 비용이 커진다.

---

# 기억해야 할 내용

- User와 Group은 사람 중심의 접근 관리 수단이다.
- Group은 자격 증명이 없고 공통 직무 권한을 묶는다.
- Workload에는 User가 아니라 Role을 사용한다.
- Root user는 MFA로 보호하고 일상 작업에 사용하지 않는다.
- 장기 액세스 키는 예외로 만들고 공유하지 않는다.
- 권한 수명 주기는 입사, 이동, 퇴사 절차와 연결한다.
- 공용 계정은 추적성과 회수 가능성을 없앤다.

---

# 다음 Chapter

다음 Chapter에서는 Workload와 계정 간 위임에 사용하는 IAM Role과 STS를 학습한다.

Trust Policy, Permission Policy, AssumeRole과 실행 환경별 Role 연결을 알아본다.
