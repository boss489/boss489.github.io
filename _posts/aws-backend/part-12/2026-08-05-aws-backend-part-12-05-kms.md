---
title: "Chapter 05. KMS"
permalink: /aws-backend/part-12/05-kms/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. KMS
## 암호화 Key를 통제하는 관리형 서비스

> **학습 목표**
>
> - Envelope Encryption과 Data Key 흐름을 설명할 수 있다.
> - Key Policy와 IAM Policy의 관계를 이해한다.
> - Rotation과 Grant의 용도를 구분할 수 있다.
> - 암호화와 접근 권한이 별개임을 설명할 수 있다.

---

# 침해 시나리오

공격자가 암호화된 S3 객체를 복사했다고 가정해 보자.

KMS 복호화 권한이 없다면 암호문만으로 원문을 읽기 어렵다.

그러나 애플리케이션 Role에 `kms:Decrypt`와 S3 읽기 권한이 모두 과도하게 열려 있다면 암호화는 유출을 막지 못한다.

암호화와 데이터 접근 권한은 별도 통제이며 둘을 함께 최소화해야 한다.

---

# KMS란?

AWS Key Management Service는 암호화 Key의 생성, 사용 권한, 감사와 수명 주기를 관리하는 서비스다.

KMS Key의 핵심 재료는 서비스 보호 경계 안에서 관리되며 API를 통해 암호화 작업을 요청한다.

| 유형 | 관리 주체 | 제어 범위 | 일반 용도 |
|---|---|---|---|
| AWS 소유 Key | AWS | 고객 제어가 적음 | 기본 서비스 암호화 |
| AWS 관리형 Key | AWS | 정책 제어가 제한됨 | 서비스별 기본 Key |
| 고객 관리형 Key | 고객 | 정책, 별칭, 수명 주기 | 분리와 감사가 필요한 데이터 |

고객 관리형 Key는 통제력이 큰 대신 정책, 가용성, 비용을 직접 운영한다.

---

# Envelope Encryption

대용량 데이터를 KMS API로 직접 처리하지 않고 Data Key로 데이터를 암호화하는 방식을 Envelope Encryption이라고 한다.

```
KMS Key
   │ GenerateDataKey
   ▼
Plaintext Data Key + Encrypted Data Key
   │                         │
   │ 데이터 암호화           └──▶ 암호문과 함께 저장
   ▼
Ciphertext

복호화 시 Encrypted Data Key ──▶ KMS Decrypt
                                      │
                                      ▼
                              Plaintext Data Key
```

평문 Data Key는 메모리에서 사용한 직후 제거하고 저장하지 않는다.

암호화된 Data Key는 원본 데이터 옆에 저장해도 KMS 권한 없이는 복호화할 수 없다.

---

# Data Key CLI 흐름

```bash
aws kms generate-data-key \
  --key-id alias/order-data \
  --key-spec AES_256

aws kms decrypt \
  --ciphertext-blob fileb://encrypted-data-key.bin \
  --encryption-context service=order
```

CLI 출력의 평문 Key를 로그나 셸 기록에 남기지 않으며 예시는 흐름 이해에만 사용한다.

Encryption Context는 비밀 값이 아니며 암호화와 복호화 요청이 같은 문맥을 사용하도록 묶는다.

---

# Key Policy와 IAM Policy

Key Policy는 KMS Key에 붙는 Resource Policy이며 누가 Key를 관리하고 사용할 수 있는지 결정한다.

IAM Policy는 호출 Identity가 어떤 KMS API와 Key를 사용할 수 있는지 표현한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["kms:Decrypt"],
    "Resource": "arn:aws:kms:ap-northeast-2:123456789012:key/KEY_ID",
    "Condition": {
      "StringEquals": {
        "kms:EncryptionContext:service": "order"
      }
    }
  }]
}
```

IAM Allow만 추가해 해결하려 하지 말고 Key Policy가 계정 IAM 권한을 허용하는 구조인지 확인한다.

Key 관리자와 Key 사용자를 분리해 관리 권한이 곧 데이터 복호화 권한이 되지 않게 한다.

---

# Rotation과 Grant

Rotation은 새 암호화에 사용할 Key Material을 교체하면서 기존 암호문의 복호화 가능성을 유지하는 수명 주기 기능이다.

Rotation은 이미 저장된 데이터를 자동으로 다시 암호화하는 작업과 다르다.

Grant는 AWS 서비스나 특정 Principal에 제한된 KMS 작업을 위임하는 별도 권한 메커니즘이다.

Grant에는 작업과 제약을 좁게 지정하고 사용이 끝나면 폐기한다.

Key를 삭제하면 복구 기간이 지난 뒤 데이터가 영구적으로 복호화 불가능해질 수 있다.

---

# 서비스 구조

```text
Spring Boot Task Role
   │ PutObject with SSE-KMS
   ▼
S3 ─────▶ KMS Key
│          │ Key Policy와 IAM 평가
│          ▼
└──── 암호화된 Object 저장

GetObject 시 S3 권한과 KMS 복호화 권한을 각각 평가
```

S3 읽기 권한만 있고 KMS 복호화 권한이 없으면 SSE-KMS 객체 읽기가 실패할 수 있다.

---

# Spring Boot 3.x 연동

```java
@Configuration
class KmsConfiguration {
    @Bean
    KmsClient kmsClient() {
        return KmsClient.builder()
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}

@Service
class DataKeyService {
    private final KmsClient kmsClient;

    DataKeyService(KmsClient kmsClient) {
        this.kmsClient = kmsClient;
    }

    GenerateDataKeyResponse createDataKey(String keyId) {
        return kmsClient.generateDataKey(builder -> builder
                .keyId(keyId)
                .keySpec(DataKeySpec.AES_256)
                .encryptionContext(Map.of("service", "order")));
    }
}
```

실제 구현은 평문 Data Key를 가능한 짧게 유지하고 민감 배열을 사용 후 덮어쓰는 라이브러리를 검토한다.

운영 환경에는 장기 키를 넣지 않고 Task Role과 기본 자격 증명 공급자를 사용한다.

---

# 실무 운영

환경과 데이터 등급별 Key 분리는 사고 격리와 권한 검토에 유리하다.

Alias는 애플리케이션 설정을 읽기 쉽게 하지만 실제 감사에서는 Key ID를 확인한다.

CloudTrail로 Encrypt, Decrypt, GenerateDataKey와 Key 정책 변경을 감시한다.

Key 비활성화와 삭제는 의존 서비스, Backup, 재해 복구 데이터를 확인한 뒤 승인한다.

---

# 장애와 보안 사고

`InvalidCiphertextException`은 잘못된 암호문, 다른 Key, 불일치한 Encryption Context를 확인한다.

`AccessDeniedException`은 호출 Role, Key Policy, IAM Policy, Grant와 Region을 확인한다.

Key가 비활성화되면 여러 서비스가 동시에 실패할 수 있으므로 영향 범위를 먼저 파악한다.

복호화 급증은 데이터 유출 신호일 수 있으므로 호출 주체와 대상 데이터를 즉시 조사한다.

---

# 비용 고려사항

고객 관리형 Key 유지, KMS API 요청, CloudTrail 기록과 교차 Region 구조에서 비용이 발생한다.

매 요청마다 불필요하게 Data Key를 생성하지 말고 보안 요구에 맞는 안전한 캐싱을 검토한다.

비용 때문에 서로 무관한 모든 데이터를 한 Key로 합치면 침해 범위와 운영 결합도가 커진다.

---

# 기억해야 할 내용

- Envelope Encryption은 Data Key로 데이터를 처리하고 KMS Key로 Data Key를 보호한다.
- 평문 Data Key는 저장하거나 로그에 남기지 않는다.
- Key Policy와 IAM Policy를 함께 확인한다.
- Rotation은 기존 데이터를 자동 재암호화하지 않는다.
- Grant는 제한된 KMS 작업을 위임한다.
- 암호화와 데이터 접근 권한은 별개다.
- Key 삭제는 데이터 영구 손실로 이어질 수 있다.

---

# 다음 Chapter

다음 Chapter에서는 비밀번호와 API Token의 수명 주기를 관리하는 Secrets Manager를 학습한다.

Parameter Store와의 차이, Rotation과 캐싱을 알아본다.
