---
title: "Chapter 01. Security Overview"
permalink: /aws-backend/part-12/01-security-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Security Overview
## AWS 보안의 출발점

> **학습 목표**
>
> - Shared Responsibility Model을 설명할 수 있다.
> - Defense in Depth와 최소 권한을 설계에 적용할 수 있다.
> - 장기 액세스 키를 피해야 하는 이유를 설명할 수 있다.
> - 침해 사고의 공격 경로와 방어 계층을 연결할 수 있다.

---

# 침해 시나리오

개발자가 장기 액세스 키를 소스 코드에 넣고 공개 저장소에 Push했다고 가정해 보자.

공격자는 자동화된 검색으로 키를 발견하고 허용된 API를 호출해 데이터를 복사하거나 고비용 자원을 생성한다.

권한이 `AdministratorAccess`라면 하나의 키 유출이 계정 전체 침해로 확대된다.

CloudTrail이 꺼져 있거나 로그가 보호되지 않았다면 조사에 필요한 호출 기록까지 사라질 수 있다.

이 문제는 키 하나만 교체해서 끝나는 문제가 아니라 자격 증명, 권한, 탐지, 로그, 복구 계층이 함께 실패한 결과다.

---

# Shared Responsibility Model

AWS는 클라우드 자체의 보안을 책임지고 고객은 클라우드 안에서 구성한 자원과 데이터를 책임진다.

| 구분 | AWS 책임 | 고객 책임 |
|---|---|---|
| 물리 시설 | 데이터 센터와 하드웨어 보호 | 해당 없음 |
| 관리형 서비스 기반 | 서비스 인프라와 하이퍼바이저 | 서비스 설정과 데이터 |
| 운영체제 | 관리형 범위에 따라 다름 | EC2 게스트 OS 패치 |
| 접근 제어 | IAM 서비스 제공 | 정책과 자격 증명 관리 |
| 암호화 | 암호화 기능 제공 | 적용 범위와 Key 정책 결정 |
| 감사 | CloudTrail 기능 제공 | Trail 설정과 로그 보호 |

RDS가 관리형 서비스여도 공개 접근, 약한 비밀번호, 과도한 Security Group은 고객 책임이다.

S3 내구성이 높아도 공개 정책과 잘못된 삭제 권한은 고객이 통제해야 한다.

---

# Defense in Depth

Defense in Depth는 하나의 통제가 실패해도 다른 계층이 공격을 차단하거나 탐지하도록 방어선을 겹치는 원칙이다.

```
Internet
   │
   ▼
WAF ── 악성 요청 차단
   │
   ▼
ALB + TLS ── 전송 보호
   │
   ▼
Private Subnet + Security Group ── 네트워크 제한
   │
   ▼
ECS Task Role ── 최소 권한
   │
   ▼
KMS + Resource Policy ── 데이터 보호
   │
   ▼
CloudTrail + GuardDuty ── 기록과 탐지
```

WAF만으로 애플리케이션 권한 검사를 대신할 수 없고 암호화만으로 잘못된 데이터 조회 권한을 막을 수도 없다.

예방, 탐지, 대응, 복구 통제를 함께 설계해야 운영 가능한 보안이 된다.

---

# 최소 권한

최소 권한은 주체가 현재 작업에 필요한 Action과 Resource만 필요한 조건에서 사용하게 하는 원칙이다.

개발 초기의 와일드카드 정책을 운영 환경에 그대로 배포하면 편의가 침해 범위로 변한다.

다음 정책은 특정 운영 Bucket의 주문 경로만 읽도록 제한한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::order-prod/orders/*"
  }]
}
```

정책은 배포 시점뿐 아니라 CloudTrail 사용 기록과 Access Analyzer 결과를 근거로 계속 축소해야 한다.

---

# 자격 증명 선택

| 실행 주체 | 권장 방식 | 피해야 할 방식 |
|---|---|---|
| 사람 | Identity Center 또는 IAM User와 MFA | Root 상시 사용 |
| ECS Task | ECS Task Role | 이미지 안 액세스 키 |
| EC2 애플리케이션 | Instance Profile | 환경 변수 장기 키 |
| EKS Pod | IRSA | Node Role 공유 |
| 외부 CI | OIDC 기반 Role 위임 | Repository Secret 장기 키 |

장기 액세스 키는 복제하기 쉽고 만료가 자동화되지 않으며 유출 후 사용 위치를 통제하기 어렵다.

Workload는 Role과 STS 임시 자격 증명을 사용해 수명과 권한 범위를 줄여야 한다.

Root user는 계정 복구와 제한된 계정 수준 작업에만 사용하고 MFA를 설정한 뒤 일상 작업에는 사용하지 않는다.

---

# CLI로 현재 주체 확인

작업 전에는 호출 주체를 확인해 잘못된 계정이나 Role에서 변경하는 사고를 막는다.

```bash
aws sts get-caller-identity

aws iam get-account-summary
```

`get-caller-identity` 결과의 Account와 Arn을 배포 로그에 남기면 권한 오류와 오배포 조사에 도움이 된다.

---

# Spring Boot 3.x 기본 연동

AWS SDK v2의 `DefaultCredentialsProvider`는 실행 환경에 맞는 표준 자격 증명 공급자를 순서대로 탐색한다.

운영 환경에서는 ECS Task Role, EC2 Instance Profile, EKS IRSA가 공급한 임시 자격 증명을 선택하게 한다.

```java
@Configuration
class AwsSecurityConfiguration {
    @Bean
    S3Client s3Client() {
        return S3Client.builder()
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}
```

클라이언트를 Bean으로 만들고 생성자 주입하면 연결 자원을 재사용하고 테스트에서 대체하기 쉽다.

`AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`에 장기 키를 넣어 운영 컨테이너에 배포하지 않는다.

---

# 실무 점검

계정에는 Root MFA와 대체 연락처를 설정하고 Root 액세스 키가 없는지 확인한다.

운영과 개발 계정을 분리하고 조직 수준 SCP로 명백히 위험한 작업을 제한한다.

모든 Workload에는 독립 Role을 부여하고 같은 Role을 여러 서비스가 공유하지 않게 한다.

민감 데이터는 저장과 전송 구간에서 암호화하고 복호화 권한을 별도로 제한한다.

CloudTrail 로그는 중앙 계정의 변경 불가능한 저장소에 모으고 경보와 연결한다.

---

# 장애와 보안 사고

권한을 급하게 축소한 뒤 API가 `AccessDenied`를 반환하면 요청 ID, 호출 주체, Action, Resource, 명시적 Deny 여부를 순서대로 확인한다.

키 유출이 의심되면 키를 비활성화하고 세션과 권한을 차단한 뒤 CloudTrail로 최초 사용과 영향 범위를 조사한다.

침해 대응 중 로그를 먼저 삭제하거나 인스턴스를 즉시 종료하면 증거가 사라질 수 있으므로 격리와 보존 절차를 따른다.

암호화 키를 비활성화하면 공격자뿐 아니라 정상 서비스도 데이터를 읽지 못하므로 의존 자원과 복구 절차를 확인한다.

---

# 비용 고려사항

IAM 자체의 권한 모델보다 CloudTrail 추가 이벤트, GuardDuty 분석, Config 기록, KMS API 호출과 로그 보관에서 비용이 발생한다.

비용을 줄이려고 감사 로그를 제거하면 사고의 탐지 시간과 복구 비용이 더 커질 수 있다.

데이터 이벤트와 보존 기간은 위험도와 규정에 따라 선택하고 비용 알림을 함께 구성한다.

---

# 기억해야 할 내용

- AWS와 고객의 책임 경계는 서비스 유형에 따라 달라진다.
- Defense in Depth는 예방부터 복구까지 여러 통제를 겹친다.
- 최소 권한은 한 번 설정하는 작업이 아니라 사용 기록을 기반으로 계속 줄이는 과정이다.
- 사람은 강한 인증을 사용하고 Workload는 Role을 사용한다.
- 장기 액세스 키는 피하고 STS 임시 자격 증명을 사용한다.
- Root user는 MFA로 보호하고 일상 작업에는 사용하지 않는다.
- 암호화, 접근 제어, 감사 로그는 서로 대체할 수 없다.

---

# 다음 Chapter

다음 Chapter에서는 사람의 AWS 접근을 관리하는 IAM User와 Group을 학습한다.

Root 보호, MFA, 권한 그룹화와 장기 키 제거 절차를 구체적으로 알아본다.
