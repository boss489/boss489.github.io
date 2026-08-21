---
title: "Chapter 04. Lifecycle"
permalink: /aws-backend/part-07/04-lifecycle/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Lifecycle
## 데이터 보존 정책을 자동화하는 방법

> **학습 목표**
>
> - Lifecycle 정책의 목적을 설명할 수 있다.
> - Storage Class 전환과 만료의 차이를 이해한다.
> - 현재 버전과 비현재 버전의 규칙을 구분할 수 있다.
> - 파일 보관 기간을 비용과 함께 설계할 수 있다.

---

# 왜 Lifecycle이 필요한가

서비스가 매일 생성하는 로그와 썸네일을 모두 S3 Standard에 무기한 저장한다고 가정한다.

처음에는 작았던 데이터도 시간이 지나면 대부분 조회하지 않는 오래된 객체가 되고 저장 비용은 계속 증가한다.

Versioning을 활성화한 Bucket에서는 화면에 보이지 않는 비현재 버전까지 누적된다.

대용량 업로드가 중단된 뒤 남은 Multipart 조각도 자동으로 사라지지 않아 비용을 발생시킬 수 있다.

사람이 매일 오래된 객체를 찾아 옮기거나 삭제하는 대신 데이터 보존 규칙을 Lifecycle로 자동화해야 한다.

---

# Lifecycle이란?

S3 Lifecycle은 객체의 나이, prefix 또는 태그 조건에 따라 Storage Class를 전환하거나 객체를 만료시키는 Bucket 정책이다.

**전환(Transition)** 은 데이터를 다른 Storage Class로 이동하고 **만료(Expiration)** 는 객체 삭제를 요청한다.

Versioning Bucket에서는 현재 버전과 비현재 버전에 서로 다른 규칙을 적용할 수 있다.

Lifecycle 작업은 비동기로 수행되므로 만료 시각에 정확히 객체가 사라지는 스케줄러로 사용해서는 안 된다.

---

# 동작 흐름

로그 데이터의 접근 빈도가 시간에 따라 감소하는 흐름은 다음과 같다.

```
Object 생성
    │
    ▼
S3 Standard
    │ 일정 기간 후 전환
    ▼
저빈도 또는 Archive Class
    │ 보존 기간 종료
    ▼
Expiration
```

1. 애플리케이션이 `logs/` prefix에 객체를 저장한다.
2. S3가 Lifecycle 필터와 객체의 나이를 평가한다.
3. 전환 조건을 만족하면 지정한 Storage Class로 이동한다.
4. 만료 조건을 만족하면 현재 객체 또는 비현재 버전을 정리한다.
5. 별도 규칙은 미완료 Multipart Upload를 지정한 기간 이후 중단한다.

규칙은 활성화 전에 대상 객체, 복구 요구와 법적 보존 기간을 검토해야 한다.

---

# Storage Class 비교

| 클래스 계열 | 접근 특성 | 검색 특성 | 대표 용도 |
|-------------|-----------|-----------|-----------|
| S3 Standard | 자주 접근 | 즉시 접근 | 활성 이미지와 콘텐츠 |
| Standard-IA | 드물게 접근 | 즉시 접근, 검색 비용 고려 | 백업과 오래된 원본 |
| One Zone-IA | 드물게 접근 | 단일 AZ 저장 특성 고려 | 재생성 가능한 데이터 |
| Glacier Instant Retrieval | 매우 드문 접근 | 즉시 접근 | 즉시 읽어야 하는 아카이브 |
| Glacier Flexible Retrieval | 아카이브 | 복원 대기 필요 | 장기 백업 |
| Glacier Deep Archive | 가장 드문 접근 | 더 긴 복원 대기 | 장기 보존 |
| Intelligent-Tiering | 패턴이 불명확 | 접근 계층 자동화 | 접근 예측이 어려운 데이터 |

IA와 Glacier 계열은 검색 비용, 최소 보관 기간과 최소 과금 객체 크기 같은 제약을 반드시 확인한다.

---

# AWS CLI에서는

현재 버전 전환, 비현재 버전 만료와 미완료 Multipart 정리를 포함한 정책을 작성한다.

```json
{
  "Rules": [
    {
      "ID": "archive-logs",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 365 },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

정책을 적용하고 다시 조회한다.

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-service-bucket \
  --lifecycle-configuration file://lifecycle.json

aws s3api get-bucket-lifecycle-configuration \
  --bucket my-service-bucket
```

---

# Spring Boot에서는 어떻게 쓰는가

Lifecycle은 보통 애플리케이션 실행 때마다 변경하지 않고 IaC나 배포 파이프라인에서 관리한다.

SDK v2로 설정해야 한다면 전용 관리 서비스에서 명시적으로 구성한다.

```java
@Service
public class BucketLifecycleService {

    private final S3Client s3Client;

    public BucketLifecycleService(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    public void configure(String bucket) {
        LifecycleRule rule = LifecycleRule.builder()
                .id("expire-temporary-files")
                .status(ExpirationStatus.ENABLED)
                .filter(LifecycleRuleFilter.builder()
                        .prefix("temporary/")
                        .build())
                .expiration(LifecycleExpiration.builder()
                        .days(1)
                        .build())
                .abortIncompleteMultipartUpload(
                        AbortIncompleteMultipartUpload.builder()
                                .daysAfterInitiation(7)
                                .build())
                .build();

        s3Client.putBucketLifecycleConfiguration(request -> request
                .bucket(bucket)
                .lifecycleConfiguration(configuration -> configuration
                        .rules(rule)));
    }
}
```

애플리케이션 Role에 Bucket 관리 권한을 주는 것은 위험하므로 일반 업로드 서비스와 관리 권한을 분리한다.

---

# 실무에서는 어떻게 사용할까

원본 이미지, 파생 이미지, 임시 파일과 감사 로그는 복구 가치가 다르므로 prefix나 Bucket을 나누어 별도 규칙을 적용한다.

정책을 정할 때 데이터 소유 부서와 보존 기간, 예상 조회 시점, 허용 복원 시간을 먼저 합의한다.

삭제 예정 데이터를 태그로 표시한 뒤 Lifecycle 필터를 적용하면 업무 상태와 정리 정책을 연결할 수 있다.

규칙 변경은 기존 객체에도 영향을 줄 수 있으므로 대상 객체 수와 예상 동작을 검토한 뒤 단계적으로 적용한다.

---

# 장애 사례와 주의할 점

prefix를 `log/` 대신 빈 값으로 설정하면 Bucket 전체 객체가 만료 대상이 되는 대형 사고가 발생할 수 있다.

아카이브 클래스로 전환한 객체를 즉시 읽는 API가 그대로 남아 있으면 복원 대기 때문에 사용자 요청이 실패한다.

최소 보관 기간보다 빨리 삭제하거나 전환하면 조기 삭제 비용이 생길 수 있다.

Versioning Bucket에서 현재 버전 만료만 설정하면 Delete Marker와 비현재 버전이 계속 남을 수 있다.

---

# 비용과 성능 고려사항

Lifecycle 전환 요청, 저장 용량, 데이터 검색과 복원, 최소 보관 기간이 총비용에 영향을 준다.

작고 수명이 짧은 객체는 전환 비용과 제약 때문에 Standard에 두고 바로 만료하는 편이 나을 수 있다.

접근 패턴을 예측하기 어려우면 Intelligent-Tiering의 모니터링 비용과 자동 계층화를 비교한다.

CloudFront 캐시가 있는 콘텐츠도 원본의 보존 정책과 캐시 TTL을 별도로 설계해야 한다.

---

# 기억해야 할 내용

- Lifecycle은 전환과 만료를 자동화하는 데이터 정책이다.
- 규칙은 prefix나 태그로 대상을 제한할 수 있다.
- 현재 버전과 비현재 버전의 정리 규칙은 별개이다.
- 미완료 Multipart Upload 정리 규칙을 함께 둔다.
- 아카이브 클래스는 복원 시간과 검색 비용을 고려한다.
- 최소 보관 기간과 전환 요청 비용을 확인한다.

---

# 다음 Chapter

다음 Chapter에서는 서버가 제한된 시간의 접근 권한을 위임하는 **Presigned URL**을 학습한다.

브라우저가 Spring Boot 서버를 경유하지 않고 S3에 직접 업로드하는 흐름을 구현한다.


