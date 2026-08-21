---
title: "Chapter 06. Secrets Manager"
permalink: /aws-backend/part-12/06-secrets-manager/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Secrets Manager
## 비밀 정보의 저장과 교체

> **학습 목표**
>
> - Secrets Manager와 Parameter Store를 선택할 수 있다.
> - Rotation 단계와 실패 영향을 설명할 수 있다.
> - 비밀 정보 캐싱과 로그 보호를 설계할 수 있다.
> - Spring Boot에서 최소 권한으로 Secret을 조회할 수 있다.

---

# 침해 시나리오

데이터베이스 비밀번호가 `application-prod.yml`에 들어갔다고 가정해 보자.

Git 기록, CI 로그, 빌드 이미지와 개발자 노트북에 같은 비밀이 복제된다.

비밀번호를 바꾸면 모든 복제본과 애플리케이션을 동시에 갱신해야 해 교체가 미뤄진다.

Secrets Manager에 중앙 저장하고 실행 Role로 조회하면 배포물에서 비밀을 분리하고 Rotation을 자동화할 수 있다.

---

# Secrets Manager란?

Secrets Manager는 데이터베이스 비밀번호, API Token 같은 비밀의 저장, 버전, 접근 제어와 Rotation을 관리한다.

Secret 값은 KMS로 암호화되지만 호출자는 `GetSecretValue`와 필요한 KMS 권한을 모두 가져야 할 수 있다.

암호화가 되어 있다는 사실이 과도한 읽기 권한을 안전하게 만들지는 않는다.

---

# Parameter Store와 비교

| 기준 | Secrets Manager | Systems Manager Parameter Store |
|---|---|---|
| 중심 목적 | 비밀 수명 주기 | 계층형 설정과 파라미터 |
| Rotation | 관리 기능 제공 | 별도 자동화 필요 |
| 버전 단계 | `AWSCURRENT`, `AWSPREVIOUS` | 버전과 Label |
| 값 형태 | JSON 또는 문자열 Secret | String, StringList, SecureString |
| 선택 기준 | 자동 교체가 필요한 자격 증명 | 일반 설정과 단순 보안 설정 |

가격과 기능은 변경될 수 있으므로 현재 Region의 공식 문서를 확인하고 호출량과 저장 수를 기준으로 비교한다.

단순 기능 플래그를 Secrets Manager에 넣거나 회전할 비밀번호를 평문 Parameter에 넣지 않는다.

---

# 조회 구조

```text
Spring Boot
   │ Task Role 임시 자격 증명
   ▼
Secrets Manager
   │ IAM + Resource Policy
   ▼
KMS Decrypt
   │
   ▼
Secret Version AWSCURRENT
   │
   └──▶ 짧은 수명의 애플리케이션 캐시
```

Secret은 시작 시 한 번만 읽을지 실행 중 갱신할지 Rotation 주기와 장애 허용 범위로 결정한다.

---

# Secret 생성과 조회

```bash
aws secretsmanager create-secret \
  --name prod/order/database \
  --kms-key-id alias/secrets-prod \
  --secret-string '{"username":"order_app","password":"REDACTED"}'

aws secretsmanager get-secret-value \
  --secret-id prod/order/database \
  --version-stage AWSCURRENT
```

실제 명령에서는 비밀을 명령행 인수와 Shell History에 남기지 않도록 안전한 입력 방식을 사용한다.

CLI 출력도 터미널 공유, CI 로그, 티켓에 붙이지 않는다.

---

# 최소 권한 정책

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

Secret ARN에는 서비스가 붙이는 접미사가 있을 수 있으므로 실제 ARN을 기준으로 Resource를 작성한다.

목록 조회, 수정, 삭제, Rotation 권한은 읽기 전용 애플리케이션 Role에 주지 않는다.

---

# Rotation

Rotation은 새 비밀 생성, 대상 시스템 반영, 새 비밀 검증, 현재 버전 전환의 단계로 진행된다.

```
createSecret
    │
    ▼
setSecret
    │
    ▼
testSecret
    │
    ▼
finishSecret
```

각 단계는 재시도되어도 안전하도록 멱등하게 구현한다.

애플리케이션이 이전 값과 새 값을 전환 기간에 처리할 수 있는지 확인한다.

Rotation 함수 Role에는 대상 Secret과 데이터베이스에 필요한 최소 권한만 부여한다.

---

# Spring Boot 3.x 연동

```java
@Configuration
class SecretConfiguration {
    @Bean
    SecretsManagerClient secretsManagerClient() {
        return SecretsManagerClient.builder()
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}

@Service
class DatabaseSecretService {
    private final SecretsManagerClient client;

    DatabaseSecretService(SecretsManagerClient client) {
        this.client = client;
    }

    String load(String secretId) {
        return client.getSecretValue(builder -> builder.secretId(secretId))
                .secretString();
    }
}
```

반환 문자열 자체를 로그에 남기지 않고 필요한 필드로 Parse한 뒤 노출 범위를 최소화한다.

Spring Cloud AWS의 Secrets Manager Property Source는 설정 통합을 단순화하는 선택지지만 갱신 시점과 실패 동작을 확인해야 한다.

SDK 직접 사용은 조회와 캐시 정책을 명시적으로 제어해야 할 때 적합하다.

환경 변수에는 Secret 이름만 넣을 수 있지만 장기 AWS 액세스 키나 실제 비밀 값은 넣지 않는다.

---

# 캐싱

매 요청마다 Secret을 조회하면 지연, API 의존성과 비용이 증가한다.

메모리 캐시는 짧은 TTL, 동시 갱신 제어, 실패 시 마지막 정상 값 사용 정책을 가져야 한다.

TTL이 너무 길면 Rotation 뒤 이전 자격 증명을 계속 사용해 인증 장애가 발생한다.

캐시 객체의 `toString`, 예외 메시지, Heap Dump에도 비밀이 노출될 수 있음을 고려한다.

---

# 실무 운영

Secret 이름에는 환경과 서비스 경계를 포함하되 값 자체를 이름에 넣지 않는다.

읽기 Role과 Rotation 관리 Role을 분리한다.

CloudTrail에서 Secret 값 조회와 정책 변경을 감시하되 Secret 값 자체는 기록하지 않는다.

삭제는 복구 기간과 의존 애플리케이션을 확인하고 긴급 삭제를 일상 절차로 사용하지 않는다.

---

# 장애와 보안 사고

Rotation 직후 인증 실패가 늘면 `AWSCURRENT` 단계, 대상 DB 반영, 애플리케이션 캐시를 확인한다.

KMS 권한 오류는 Secrets Manager Allow만 추가해서 해결되지 않을 수 있다.

비밀 로그 노출이 발견되면 로그 접근을 차단하고 값을 교체한 뒤 복제된 저장소와 조회자를 조사한다.

Secret 조회 장애에 대비해 재시도 Backoff와 제한된 캐시 사용을 설계하되 무기한 오래된 값을 사용하지 않는다.

---

# 비용 고려사항

저장한 Secret 수, API 호출, Rotation 실행, KMS와 감사 로그가 비용에 영향을 준다.

캐싱은 비용과 지연을 줄이지만 교체 반영 지연이라는 보안 비용을 만든다.

일반 설정과 회전 없는 값을 Parameter Store로 분리하면 목적과 비용을 명확히 할 수 있다.

---

# 기억해야 할 내용

- Secrets Manager는 비밀의 저장, 버전과 Rotation을 관리한다.
- Parameter Store는 일반 설정과 계층형 파라미터에도 적합하다.
- Rotation 단계는 멱등하고 검증 가능해야 한다.
- Secret 값과 CLI 출력은 절대 로그에 남기지 않는다.
- 캐시는 지연과 비용을 낮추지만 Rotation 반영을 늦춘다.
- 애플리케이션 읽기 Role과 Rotation Role을 분리한다.
- 장기 AWS 키를 환경 변수에 저장하지 않는다.

---

# 다음 Chapter

다음 Chapter에서는 지금까지의 권한과 비밀 관리를 Spring Boot 애플리케이션에 통합한다.

SDK Client 구성, 자격 증명 공급자와 안전한 설정 경계를 알아본다.
