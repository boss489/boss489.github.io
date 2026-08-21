---
title: "Chapter 03. Versioning"
permalink: /aws-backend/part-07/03-versioning/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Versioning
## 덮어쓰기와 삭제에서 객체를 복구하는 방법

> **학습 목표**
>
> - Versioning의 목적과 상태를 설명할 수 있다.
> - 삭제와 덮어쓰기 복구 흐름을 이해한다.
> - Delete Marker와 Version ID를 설명할 수 있다.
> - Lifecycle과 함께 비용을 관리할 수 있다.

---

# 왜 Versioning이 필요한가

운영자가 상품 이미지를 교체하려다 잘못된 파일을 같은 Key에 업로드했다고 가정한다.

Versioning이 꺼져 있으면 기존 객체는 즉시 덮어써지고 애플리케이션에는 되돌릴 사본이 없다.

사용자나 배치 작업이 객체를 실수로 삭제하는 경우도 동일하다.

백업 시스템을 별도로 만들 수 있지만 변경마다 복사본을 생성하고 복구 경로를 운영해야 한다.

S3 Versioning은 같은 Key의 변경 이력을 보존하여 이러한 실수의 복구 지점을 제공한다.

---

# Versioning이란?

Versioning은 같은 Object Key에 여러 버전을 Version ID로 구분해 보관하는 Bucket 기능이다.

한 번 활성화한 Bucket은 비활성 상태로 되돌리는 대신 **Suspended** 상태로 전환할 수 있다.

Versioning 활성화 이후 생성된 객체에는 고유한 Version ID가 부여된다.

기존 객체는 Versioning을 켰다고 자동 복제되지 않으며 이후 변경부터 버전 이력이 쌓인다.

Versioning은 가용성을 대신하는 기능이 아니라 논리적인 삭제와 덮어쓰기에서 복구 가능성을 높이는 기능이다.

---

# 동작 흐름

같은 Key를 반복해서 업로드하면 최신 버전이 기본 조회 결과가 된다.

```
PUT logo.png (v1)
       │
PUT logo.png (v2)
       │
DELETE logo.png
       ▼
Key: logo.png
├── Delete Marker  ← 현재 버전
├── Version v2
└── Version v1
```

1. 첫 업로드가 `v1` Version ID로 저장된다.
2. 같은 Key를 다시 업로드하면 `v2`가 추가되고 기본 조회 대상이 된다.
3. Version ID 없이 삭제하면 객체 데이터 대신 Delete Marker가 현재 버전으로 추가된다.
4. 기본 `GetObject`는 삭제된 것처럼 응답하지만 이전 데이터 버전은 남아 있다.
5. Delete Marker를 Version ID와 함께 삭제하면 이전 버전이 다시 현재 객체로 보인다.

특정 데이터 버전을 영구 삭제하려면 해당 Version ID를 명시해 삭제해야 한다.

---

# Versioning 상태 비교

| 상태 | 새 객체의 버전 | 덮어쓰기 보존 | 특징 |
|------|----------------|---------------|------|
| 미설정 | `null` | 보존하지 않음 | 같은 Key 쓰기가 교체 |
| Enabled | 고유 Version ID | 보존함 | 삭제 시 Delete Marker 생성 |
| Suspended | `null` 버전 처리 | 기존 버전 유지 | 새 변경의 보호가 제한됨 |

MFA Delete는 추가 보호 수단이지만 일반 애플리케이션 흐름보다는 높은 통제가 필요한 운영 환경에서 검토한다.

---

# AWS CLI에서는

Bucket Versioning을 활성화하고 상태를 확인한다.

```bash
aws s3api put-bucket-versioning \
  --bucket my-service-bucket \
  --versioning-configuration Status=Enabled

aws s3api get-bucket-versioning \
  --bucket my-service-bucket
```

객체의 모든 버전과 Delete Marker를 조회한다.

```bash
aws s3api list-object-versions \
  --bucket my-service-bucket \
  --prefix products/42/image.png
```

특정 버전을 내려받거나 복구할 수 있다.

```bash
aws s3api get-object \
  --bucket my-service-bucket \
  --key products/42/image.png \
  --version-id VERSION_ID \
  ./recovered.png

aws s3api copy-object \
  --bucket my-service-bucket \
  --key products/42/image.png \
  --copy-source "my-service-bucket/products/42/image.png?versionId=VERSION_ID"
```

---

# Spring Boot에서는 어떻게 쓰는가

복구 기능은 목록에서 Version ID를 선택한 뒤 과거 버전을 현재 Key로 복사하는 방식으로 구현할 수 있다.

```java
@Service
public class ObjectVersionService {

    private final S3Client s3Client;
    private final String bucket;

    public ObjectVersionService(
            S3Client s3Client,
            @Value("${aws.s3.bucket}") String bucket) {
        this.s3Client = s3Client;
        this.bucket = bucket;
    }

    public List<ObjectVersion> findVersions(String key) {
        ListObjectVersionsRequest request = ListObjectVersionsRequest.builder()
                .bucket(bucket)
                .prefix(key)
                .build();

        return s3Client.listObjectVersionsPaginator(request)
                .versions()
                .stream()
                .filter(version -> version.key().equals(key))
                .toList();
    }

    public void restore(String key, String versionId) {
        CopyObjectRequest request = CopyObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .copySource(bucket + "/" + key + "?versionId=" + versionId)
                .build();

        s3Client.copyObject(request);
    }
}
```

복구 API는 관리자 권한과 감사 로그를 적용하고 사용자가 임의 Version ID를 넣어 다른 객체를 복구하지 못하게 검증한다.

---

# 실무에서는 어떻게 사용할까

계약서, 원본 이미지와 배포 아티팩트처럼 덮어쓰기나 삭제 복구가 중요한 Bucket에 적용한다.

Version ID를 DB에 함께 기록하면 업무 이벤트가 참조한 정확한 객체 버전을 추적할 수 있다.

복구 절차는 콘솔 조작에만 의존하지 말고 대상 Key 확인, 복구, 검증과 감사 기록 순서로 문서화한다.

[Lifecycle](/aws-backend/part-07/04-lifecycle/)에서 오래된 비현재 버전과 Delete Marker의 정리 시점을 함께 정한다.

---

# 장애 사례와 주의할 점

Versioning을 켠 뒤 Lifecycle 없이 운영하면 동일한 대용량 객체의 모든 과거 버전이 쌓여 저장 비용이 급증한다.

삭제 요청이 성공했다는 이유로 데이터가 즉시 사라졌다고 판단하면 개인정보 파기 요구를 충족하지 못할 수 있다.

완전 삭제가 필요하면 모든 데이터 버전과 Delete Marker를 대상으로 별도 삭제 절차를 수행해야 한다.

Versioning은 리전 장애나 계정 탈취에 대한 완전한 백업이 아니므로 필요하면 복제, Object Lock과 별도 계정 백업을 검토한다.

---

# 비용과 성능 고려사항

현재 버전뿐 아니라 비현재 버전도 각각 저장 용량으로 과금된다.

버전 목록 조회, 복사와 삭제 역시 요청 수에 포함된다.

비현재 버전을 저빈도 또는 아카이브 클래스로 전환할 때는 검색 비용과 최소 보관 기간을 함께 고려한다.

보존 기간을 무조건 짧게 하면 복구 가치가 사라지고 무기한으로 두면 비용이 계속 증가하므로 업무의 복구 목표에 맞춰야 한다.

---

# 기억해야 할 내용

- Versioning은 같은 Key의 여러 변경 이력을 Version ID로 보관한다.
- 일반 삭제는 Delete Marker를 추가하며 이전 데이터 버전은 남을 수 있다.
- 특정 버전의 영구 삭제에는 Version ID가 필요하다.
- 활성화한 Versioning은 끄는 대신 Suspended로 전환한다.
- Versioning은 백업 전체를 대체하지 않는다.
- 비현재 버전은 Lifecycle로 보존 기간과 비용을 관리한다.

---

# 다음 Chapter

다음 Chapter에서는 객체와 비현재 버전을 자동으로 전환하거나 만료시키는 **Lifecycle**을 학습한다.

데이터의 접근 빈도와 보존 요구를 비용 정책으로 바꾸는 방법을 살펴본다.


