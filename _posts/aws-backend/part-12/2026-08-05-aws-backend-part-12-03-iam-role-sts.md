---
title: "Chapter 03. IAM Role과 STS"
permalink: /aws-backend/part-12/03-iam-role-sts/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. IAM Role과 STS
## 임시 자격 증명으로 안전하게 위임하는 방법

> **학습 목표**
>
> - Trust Policy와 Permission Policy를 구분할 수 있다.
> - AssumeRole과 STS 임시 자격 증명 흐름을 설명할 수 있다.
> - ECS, EC2, EKS에 올바른 Role을 연결할 수 있다.
> - Role 위임 실패와 자격 증명 유출에 대응할 수 있다.

---

# 권한 문제 시나리오

Spring Boot 컨테이너 이미지에 IAM User의 장기 키가 포함되었다고 가정해 보자.

이미지를 내려받을 수 있는 사람은 누구나 키를 복사해 컨테이너 밖에서도 같은 권한을 사용할 수 있다.

키를 교체하려면 이미지와 모든 실행 Task를 다시 배포해야 하며 유출된 키에는 자동 만료도 없다.

ECS Task Role을 사용하면 Task가 필요한 시점에 STS 임시 자격 증명을 받고 만료 전에 자동 갱신한다.

---

# IAM Role이란?

Role은 고정 암호나 액세스 키를 소유하지 않고 신뢰받은 주체가 맡아 사용하는 IAM Identity다.

Role은 누가 맡을 수 있는지와 맡은 뒤 무엇을 할 수 있는지를 별도 정책으로 표현한다.

| 정책 | 질문 | 핵심 요소 |
|---|---|---|
| Trust Policy | 누가 이 Role을 맡는가 | Principal, sts:AssumeRole, Condition |
| Permission Policy | 맡은 주체가 무엇을 하는가 | Action, Resource, Condition |

Trust Policy가 허용해도 Permission Policy에 S3 권한이 없다면 S3를 읽을 수 없다.

Permission Policy가 넓어도 신뢰받지 않은 Principal은 Role을 맡을 수 없다.

---

# AssumeRole과 STS

AWS Security Token Service는 Role 세션에 사용할 임시 Access Key ID, Secret Access Key, Session Token을 발급한다.

```
Caller
  │ sts:AssumeRole
  ▼
Role Trust Policy ── Principal과 Condition 검사
  │ 허용
  ▼
STS Temporary Credentials
  │ AccessKey + SecretKey + SessionToken + Expiration
  ▼
AWS API
  │
  └──▶ Role Permission Policy와 전체 정책 평가
```

임시 자격 증명은 만료되므로 유출 시 공격 가능 시간이 장기 키보다 짧다.

세션 이름, Source Identity, Session Tag를 사용하면 CloudTrail에서 실제 호출자를 추적하기 쉽다.

---

# Trust Policy 예시

다음 Trust Policy는 특정 계정의 배포 Role만 운영 Role을 맡게 한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111122223333:role/deploy-role"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "deployment-pipeline"
      }
    }
  }]
}
```

`Principal`은 이 정책이 적용될 Role이 아니라 Role을 맡도록 신뢰하는 주체다.

제3자 SaaS에 계정 Role을 위임할 때는 추측하기 어려운 External ID로 Confused Deputy 위험을 줄인다.

---

# Permission Policy 예시

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue"
    ],
    "Resource": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/order-db-*",
    "Condition": {
      "StringEquals": {
        "aws:ResourceTag/Environment": "prod"
      }
    }
  }]
}
```

Role 정책은 서비스 전체 와일드카드보다 환경, ARN, Tag 조건으로 범위를 좁힌다.

---

# CLI AssumeRole

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/operations-role \
  --role-session-name incident-20260805 \
  --external-id deployment-pipeline

aws sts get-caller-identity
```

반환된 세 자격 증명 중 Session Token을 누락하면 API 인증이 실패한다.

운영자는 임시 값을 터미널 기록이나 채팅에 남기지 말고 CLI Profile과 SSO를 사용한다.

---

# 실행 환경별 Role

| 환경 | 연결 방식 | 자격 증명 범위 |
|---|---|---|
| ECS | ECS Task Role | 개별 Task와 애플리케이션 |
| EC2 | Instance Profile | 인스턴스의 프로세스 |
| EKS | IRSA | Kubernetes ServiceAccount와 Pod |

ECS의 Task Execution Role은 이미지 Pull과 로그 전송에 사용되고 Task Role은 애플리케이션 API 호출에 사용된다.

EC2 Instance Profile은 IAM Role을 EC2에 연결하는 컨테이너이며 애플리케이션은 메타데이터 경로로 임시 자격 증명을 얻는다.

EKS IRSA는 OIDC와 ServiceAccount를 연결해 Node Role을 모든 Pod가 공유하는 문제를 줄인다.

---

# Spring Boot 3.x 연동

AWS SDK v2는 실행 환경의 임시 자격 증명을 `DefaultCredentialsProvider`로 자동 탐색하고 갱신한다.

```java
@Configuration
class StorageConfiguration {
    @Bean
    S3Client s3Client() {
        return S3Client.builder()
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}

@Service
class OrderArchiveService {
    private final S3Client s3Client;

    OrderArchiveService(S3Client s3Client) {
        this.s3Client = s3Client;
    }
}
```

환경 변수에 장기 키를 넣으면 기본 공급자 체인이 Role보다 그 값을 먼저 선택할 수 있으므로 운영 환경에서 제거한다.

---

# 실무 운영

Role은 서비스와 환경별로 분리해 한 애플리케이션 침해가 다른 서비스로 번지는 것을 막는다.

교차 계정 Role에는 Organization ID, External ID, Source ARN 같은 조건을 적용한다.

Role 세션의 최대 시간은 작업에 필요한 범위로 정하고 장시간 작업은 안전한 갱신을 지원해야 한다.

CloudTrail에서 AssumeRole 이벤트와 그 세션의 후속 API 호출을 함께 추적한다.

---

# 장애와 보안 사고

`AccessDenied`가 발생하면 Caller의 `sts:AssumeRole` 권한과 대상 Role의 Trust Policy를 모두 확인한다.

ECS에서 자격 증명을 찾지 못하면 Task Role 연결과 메타데이터 접근 상태를 확인하고 이미지에 키를 추가하지 않는다.

EKS Pod가 Node Role 권한을 사용하면 IRSA Annotation, OIDC Provider, SDK 버전과 환경 변수를 확인한다.

임시 자격 증명이 유출되면 대상 Role 정책을 축소하거나 세션 차단 통제를 적용하고 CloudTrail로 사용 범위를 조사한다.

---

# 비용 고려사항

STS와 Role 선택은 장기 키 교체 자동화와 사고 범위를 줄여 운영 비용을 낮춘다.

과도한 AssumeRole 호출은 지연과 API 사용량을 늘릴 수 있으므로 SDK의 자격 증명 캐싱과 자동 갱신을 활용한다.

계정 분리와 감사 로그 보관 비용은 침해 격리와 추적 가능성의 대가로 평가한다.

---

# 기억해야 할 내용

- Role은 Workload와 권한 위임에 사용하는 임시 Identity다.
- Trust Policy는 누가 맡는지 결정하고 Permission Policy는 무엇을 하는지 결정한다.
- STS는 만료되는 세 가지 자격 증명을 발급한다.
- ECS는 Task Role, EC2는 Instance Profile, EKS는 IRSA를 사용한다.
- ECS Task Role과 Task Execution Role의 책임은 다르다.
- 장기 키를 이미지나 환경 변수에 넣지 않는다.
- 세션 식별 정보를 남겨 실제 호출자를 추적한다.

---

# 다음 Chapter

다음 Chapter에서는 여러 정책이 결합될 때 최종 권한을 계산하는 IAM Policy 평가를 학습한다.

Identity Policy, Resource Policy, Permissions Boundary와 SCP의 차이를 알아본다.
