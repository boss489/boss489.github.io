---
title: "Chapter 04. IAM Policy"
permalink: /aws-backend/part-12/04-iam-policy/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. IAM Policy
## 요청의 최종 허용 여부를 계산하는 규칙

> **학습 목표**
>
> - Policy의 Action, Resource, Condition, Principal을 정확히 설명할 수 있다.
> - Implicit Deny, Explicit Allow, Explicit Deny의 우선순위를 설명할 수 있다.
> - Identity Policy와 Resource Policy를 구분할 수 있다.
> - Permissions Boundary와 SCP를 권한 부여 정책으로 오해하지 않는다.

---

# 권한 문제 시나리오

개발자가 S3 읽기 정책을 Role에 추가했는데도 계속 `AccessDenied`가 발생한다고 가정해 보자.

조사 결과 조직 SCP가 운영 데이터 접근을 거부하거나 Bucket Policy에 명시적 Deny가 있을 수 있다.

반대로 Permissions Boundary에 Action이 있어도 Identity Policy가 Allow하지 않으면 요청은 허용되지 않는다.

정책 한 장만 읽어서는 결과를 알 수 없으며 요청 컨텍스트에 적용되는 전체 정책을 평가해야 한다.

---

# Policy 구성 요소

IAM Policy는 JSON Statement로 AWS API 요청에 대한 Effect를 선언한다.

| 요소 | 의미 | 예시 |
|---|---|---|
| Action | 허용하거나 거부할 API 작업 | `s3:GetObject` |
| Resource | 작업 대상 ARN | `arn:aws:s3:::order-prod/*` |
| Condition | 허용 조건 | TLS, Tag, Source ARN |
| Principal | 정책이 적용할 주체 | Role, Account, Service |

`Principal`은 주로 Resource Policy와 Role Trust Policy에서 사용하며 Identity Policy에는 일반적으로 쓰지 않는다.

지원하는 Action, Resource 수준, Condition Key는 서비스별 권한 문서에서 확인해야 한다.

---

# 평가 원칙

모든 요청은 기본적으로 Implicit Deny 상태에서 시작한다.

적용되는 정책 중 하나가 Explicit Allow하고 어떤 정책에도 Explicit Deny가 없어야 요청이 허용된다.

하나라도 Explicit Deny가 있으면 다른 Allow보다 우선한다.

```
Request
   │
   ▼
적용 정책 수집
   │
   ├── Explicit Deny 존재 ──▶ Deny
   │
   ├── Explicit Allow 존재 ─▶ Allow
   │
   └── 아무 Allow 없음 ─────▶ Implicit Deny
```

인증되지 않은 요청, 세션 정책, VPC Endpoint Policy 같은 요소도 서비스와 요청에 따라 결과에 참여한다.

---

# 최소 권한 정책

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOrderPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::order-prod",
      "Condition": {
        "StringLike": {
          "s3:prefix": "orders/*"
        }
      }
    },
    {
      "Sid": "ReadOrderObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::order-prod/orders/*"
    }
  ]
}
```

Bucket 자체 작업과 Object 작업은 Resource ARN 형태가 다르므로 분리한다.

---

# 보안 조건

다음 Resource Policy Statement는 TLS를 사용하지 않는 S3 요청을 명시적으로 거부한다.

```json
{
  "Sid": "DenyInsecureTransport",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::order-prod",
    "arn:aws:s3:::order-prod/*"
  ],
  "Condition": {
    "Bool": {
      "aws:SecureTransport": "false"
    }
  }
}
```

`Principal: "*"`는 공개 허용이 아니라 이 Deny Statement를 모든 주체에 적용한다는 의미다.

---

# 정책 유형 비교

| 정책 | 연결 위치 | 역할 | 권한을 직접 부여하는가 |
|---|---|---|---|
| Identity Policy | User, Group, Role | Identity가 할 수 있는 작업 | 그렇다 |
| Resource Policy | S3, KMS 등 Resource | 누가 Resource에 접근하는지 | 그렇다 |
| Permissions Boundary | User, Role | Identity 권한의 최대 경계 | 아니다 |
| SCP | Organization 계정, OU | 멤버 계정 권한의 최대 경계 | 아니다 |

Permissions Boundary와 SCP는 가능한 권한의 상한이며 그 안의 작업을 별도 Allow 없이 허용하지 않는다.

SCP는 관리 계정과 일부 서비스 연결 역할 등 적용 범위를 정확히 확인해야 한다.

KMS Key Policy처럼 Resource Policy가 핵심 권한 경로인 서비스도 있으므로 서비스 특성을 반영한다.

---

# CLI 검증

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/order-api-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::order-prod/orders/100.json

aws accessanalyzer validate-policy \
  --policy-type IDENTITY_POLICY \
  --policy-document file://order-policy.json
```

시뮬레이터는 유용하지만 실제 Resource Policy, SCP와 실행 컨텍스트를 완전히 대신하지 않으므로 CloudTrail 결과도 확인한다.

---

# Spring Boot 권한 설계

애플리케이션 코드는 AWS SDK v2와 실행 Role만 사용하고 권한 정책은 인프라 코드에서 관리한다.

```java
@Service
class OrderDocumentService {
    private final S3Client s3Client;

    OrderDocumentService(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    byte[] read(String bucket, String key) {
        var request = GetObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build();
        return s3Client.getObjectAsBytes(request).asByteArray();
    }
}
```

이 Service의 Task Role에는 특정 Bucket Prefix의 `s3:GetObject`만 허용하고 삭제 권한은 주지 않는다.

`DefaultCredentialsProvider`를 사용하고 환경 변수 장기 키로 정책 문제를 우회하지 않는다.

---

# 실무 운영

정책 이름보다 Statement의 실제 Action, Resource, Condition을 코드 리뷰한다.

CloudTrail의 사용 기록과 Access Advisor를 이용해 장기간 사용하지 않은 권한을 제거한다.

배포 전에 Policy Validation과 테스트 계정의 실제 API 호출을 함께 수행한다.

긴급 권한은 별도 Role로 만들고 승인과 만료 절차를 둔다.

---

# 장애와 보안 사고

권한 오류는 Caller ARN, Action, Resource ARN, 요청 Region, 암호화 Key ARN을 먼저 기록한다.

Explicit Deny는 오류 메시지에 정책 유형이 표시될 수 있지만 항상 모든 세부 정보가 노출되는 것은 아니다.

Resource의 오타와 Object ARN 누락은 Allow가 있어 보이면서 실패하는 흔한 원인이다.

정책 침해가 발견되면 와일드카드를 줄이고 이미 수행된 변경은 CloudTrail로 별도 복구한다.

---

# 비용 고려사항

IAM Policy 자체보다 정책 분석, CloudTrail 데이터 이벤트, Config 기록과 중앙 로그 보관에 비용이 발생한다.

세밀한 정책은 초기 설계 비용이 들지만 침해 범위와 운영 중 권한 오용을 줄인다.

모든 이벤트를 무조건 수집하기보다 위험한 Resource와 Action을 기준으로 감사 범위를 설계한다.

---

# 기억해야 할 내용

- 기본 상태는 Implicit Deny다.
- Explicit Allow가 있어야 허용된다.
- Explicit Deny는 모든 Allow보다 우선한다.
- Action은 작업, Resource는 대상, Condition은 조건, Principal은 주체다.
- Identity Policy와 Resource Policy는 권한을 부여할 수 있다.
- Permissions Boundary와 SCP는 권한의 최대 범위일 뿐 권한을 부여하지 않는다.
- 정책은 시뮬레이션과 실제 요청으로 검증한다.

---

# 다음 Chapter

다음 Chapter에서는 데이터 암호화 Key를 관리하는 KMS를 학습한다.

Envelope Encryption, Data Key, Key Policy, Rotation과 Grant를 알아본다.
