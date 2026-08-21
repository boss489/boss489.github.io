---
title: "Chapter 07. Spring Boot Security Integration"
permalink: /aws-backend/part-12/07-spring-boot-security-integration/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Spring Boot Security Integration
## 장기 키 없이 AWS 서비스를 호출하는 애플리케이션

> **학습 목표**
>
> - AWS SDK v2의 기본 자격 증명 흐름을 설명할 수 있다.
> - Secrets Manager Client를 안전하게 구성할 수 있다.
> - 생성자 주입으로 AWS 연동 책임을 분리할 수 있다.
> - Spring Cloud AWS를 선택적으로 평가할 수 있다.

---

# 침해 시나리오

운영 컨테이너의 환경 변수에 장기 Access Key와 Secret Key가 있다고 가정해 보자.

환경 조회 권한, 진단 Dump, 배포 명세 또는 오류 리포트가 키를 노출할 수 있다.

키는 컨테이너 수명과 무관하게 계속 유효하고 다른 위치에서도 사용할 수 있다.

실행 Role과 STS 임시 자격 증명을 사용하면 배포물에서 키를 제거하고 자동 갱신할 수 있다.

---

# 권장 구조

```text
ECS Task
  │
  ├── Task Role ── STS Temporary Credentials
  │
  ▼
Spring Boot 3.x
  │ DefaultCredentialsProvider
  ├──▶ Secrets Manager
  ├──▶ S3
  └──▶ KMS
```

애플리케이션은 자격 증명 값을 알지 않고 SDK 공급자에게 현재 실행 Identity를 요청한다.

---

# DefaultCredentialsProvider

AWS SDK v2의 `DefaultCredentialsProvider`는 System Property, 환경 변수, Web Identity, Profile, Container, Instance Metadata 등 표준 공급자를 탐색한다.

로컬에서는 SSO Profile을 사용하고 ECS에서는 Task Role, EC2에서는 Instance Profile, EKS에서는 IRSA를 사용한다.

환경 변수에 장기 키가 남아 있으면 실행 Role보다 먼저 선택될 수 있으므로 운영 배포에서 제거한다.

| 환경 | 권장 자격 증명 | 금지할 방식 |
|---|---|---|
| 로컬 | AWS SSO Profile | 팀 공용 키 |
| ECS | Task Role | 이미지 내 키 |
| EC2 | Instance Profile | User 액세스 키 |
| EKS | IRSA | 모든 Pod의 Node Role 공유 |
| CI | OIDC Role | 장기 Repository Secret |

---

# AWS SDK v2 설정

```java
@Configuration
class AwsClientConfiguration {
    @Bean
    SecretsManagerClient secretsManagerClient() {
        return SecretsManagerClient.builder()
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }

    @Bean
    S3Client s3Client() {
        return S3Client.builder()
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}
```

SDK Client는 Thread-safe하므로 요청마다 만들지 않고 Singleton Bean으로 재사용한다.

Region은 `@ConfigurationProperties`로 외부화하고 환경별 Profile에서 명확히 지정한다.

---

# 타입 안전한 설정

```java
@ConfigurationProperties(prefix = "app.aws")
public record AwsProperties(
        String region,
        String databaseSecretId,
        String documentBucket
) {
}
```

설정에는 Secret 값이 아니라 Secret ID와 Resource 이름만 둔다.

Bean Validation을 적용해 필수 설정 누락을 시작 시점에 탐지한다.

---

# Secrets Manager 최소 코드

```java
@Service
class DatabaseCredentialsProvider {
    private final SecretsManagerClient client;
    private final ObjectMapper objectMapper;
    private final AwsProperties properties;

    DatabaseCredentialsProvider(
            SecretsManagerClient client,
            ObjectMapper objectMapper,
            AwsProperties properties
    ) {
        this.client = client;
        this.objectMapper = objectMapper;
        this.properties = properties;
    }

    DatabaseCredentials load() throws JsonProcessingException {
        String value = client.getSecretValue(request -> request
                .secretId(properties.databaseSecretId()))
                .secretString();
        return objectMapper.readValue(value, DatabaseCredentials.class);
    }
}

record DatabaseCredentials(String username, String password) {
}
```

비밀 Record를 로그 인자로 전달하지 않고 예외에도 원문을 포함하지 않는다.

비밀번호를 반환하는 범위가 커지지 않도록 데이터 소스 구성 계층 안에서만 사용한다.

---

# 최소 IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "secretsmanager:GetSecretValue",
    "Resource": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/order/database-*"
  }]
}
```

고객 관리형 KMS Key를 사용한다면 해당 Key의 제한된 `kms:Decrypt` 권한과 Key Policy도 확인한다.

---

# Spring Cloud AWS

Spring Cloud AWS는 Parameter와 Secret을 Spring Environment에 통합하고 서비스별 Starter를 제공하는 선택지다.

의존성 버전이 Spring Boot 3.x와 호환되는지 확인하고 자동 구성의 Credential, Region, Reload 동작을 이해해야 한다.

단순한 조회 몇 개라면 SDK 직접 사용이 동작과 장애 경계를 더 명확하게 만들 수 있다.

설정 자동 Reload가 필요하다면 Rotation 시점, Bean 재생성, 실패 시 이전 값 정책을 테스트한다.

---

# 테스트

단위 테스트에서는 AWS Client 호출을 Adapter 뒤로 감추거나 Client를 Mock으로 주입한다.

통합 테스트는 로컬 고정 키를 코드에 넣지 않고 테스트 전용 Profile 또는 에뮬레이터를 사용한다.

운영과 같은 IAM 정책 검증은 별도 테스트 계정에서 실제 Role로 수행한다.

Secret 값이 테스트 실패 메시지와 Snapshot에 포함되지 않는지 확인한다.

---

# 장애와 보안 사고

자격 증명 탐색 실패 시 실행 Role 연결, Web Identity 파일, 메타데이터 접근과 공급자 우선순위를 확인한다.

`AccessDenied`는 애플리케이션에 관리자 키를 넣지 말고 Caller ARN과 정책 평가를 조사한다.

Secret 조회 지연은 모든 요청을 막을 수 있으므로 제한 시간, Backoff, Circuit Breaker와 안전한 캐시를 검토한다.

로그에 비밀이 노출되면 마스킹만 추가하지 말고 값을 즉시 교체하고 로그 복제 범위를 조사한다.

---

# 실무와 비용

서비스별 Task Role을 사용하고 AWS Client와 Secret Cache의 지표를 수집한다.

SDK 호출 횟수, Secret 조회, KMS 요청, 로그 저장과 네트워크 경로가 비용에 영향을 준다.

VPC Endpoint는 보안 경로를 단순화할 수 있지만 Endpoint 자체 비용과 정책 운영을 함께 평가한다.

캐싱은 비용을 줄이지만 Rotation 반영 시간과 메모리 노출 범위를 늘린다.

---

# 기억해야 할 내용

- AWS SDK v2는 `DefaultCredentialsProvider`를 사용한다.
- 운영 환경에서는 Role 기반 임시 자격 증명을 사용한다.
- 장기 키를 환경 변수나 설정 파일에 넣지 않는다.
- SDK Client는 Bean으로 재사용하고 생성자 주입한다.
- Secret 값은 로그와 예외에 포함하지 않는다.
- Spring Cloud AWS는 요구와 자동 구성 동작을 확인해 선택한다.
- IAM 오류를 관리자 키로 우회하지 않는다.

---

# 다음 Chapter

다음 Chapter에서는 예방 통제가 실패한 뒤 탐지하고 대응하는 Security Operations를 학습한다.

CloudTrail, Config, Access Analyzer, GuardDuty와 Security Hub의 역할을 구분한다.
