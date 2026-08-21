---
title: "Chapter 02. Bucket and Object"
permalink: /aws-backend/part-07/02-bucket-object/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Bucket and Object
## 파일 서비스의 경계와 식별자

> **학습 목표**
>
> - Bucket과 Object의 관계를 설명할 수 있다.
> - Object Key와 prefix의 의미를 이해한다.
> - S3 권한 설정의 기본 단위를 이해한다.
> - 충돌과 정보 노출을 줄이는 Key를 설계할 수 있다.

---

# 왜 Bucket과 Object 설계가 필요한가

프로필 이미지를 원본 파일명 그대로 `profile.png`라는 Key에 저장하면 다음 사용자의 업로드가 이전 파일을 덮어쓴다.

한 Bucket에 운영 파일, 임시 파일과 로그를 구분 없이 저장하면 권한과 보관 기간을 각각 적용하기 어렵다.

URL에 이메일 같은 정보를 포함한 Key는 접근 로그와 브라우저 기록에 개인정보를 남긴다.

파일 서비스의 안정성은 **Bucket 경계와 Object Key 규칙**을 어떻게 정하는지에 달려 있다.

---

# Bucket과 Object란?

Bucket은 S3 객체를 담는 최상위 컨테이너이다.

일반 목적 Bucket 이름은 AWS 파티션 안에서 전역적으로 고유해야 한다.

Object는 S3에 저장되는 파일 단위이며 Key, Data, Metadata, Version ID와 Storage Class를 가진다.

Object Key는 객체의 이름이다.

`images/2026/08/sample.png`처럼 경로처럼 보이지만 실제 디렉터리는 아니다.

S3 콘솔이 `/`를 기준으로 폴더처럼 보여 주지만 내부적으로는 평평한 Key 공간에서 prefix를 공유하는 객체들이다.

Bucket은 리전을 선택해 생성하며 객체 데이터는 선택한 리전에 저장된다.

---

# 동작 흐름

애플리케이션은 업무 식별자로 Key를 만든 뒤 Bucket과 Key의 조합으로 객체를 저장하고 조회한다.

```
Upload Request
      │
      ▼
Spring Boot
      │ Key: users/42/profile/{uuid}.png
      ▼
S3 Bucket
└── Object
    ├── Key
    ├── Data
    ├── Metadata
    └── Storage Class
```

1. 서버가 인증된 사용자와 업로드 목적을 확인한다.
2. 서버가 사용자 입력과 분리된 고유한 Key를 생성한다.
3. `PutObject` 요청에 Bucket, Key, 콘텐츠 타입과 데이터를 담는다.
4. S3는 같은 Bucket과 Key가 이미 존재하면 새 객체로 덮어쓴다.
5. 서버는 Key와 소유자 정보를 DB에 저장하고 이후 인가 판단에 사용한다.

S3의 강한 일관성 덕분에 성공한 쓰기 직후 같은 Key를 읽거나 목록에서 조회할 수 있다.

---

# Bucket과 Key 설계 비교

| 설계 | 장점 | 주의점 |
|------|------|--------|
| 환경별 Bucket 분리 | 권한과 삭제 사고 범위 분리 | 설정 관리 증가 |
| 용도별 Bucket 분리 | 원본, 로그 정책 분리 | 데이터 흐름 관리 필요 |
| 하나의 Bucket과 prefix | 중앙 관리가 단순함 | 정책 조건이 복잡해짐 |
| 원본 파일명 Key | 구현이 쉬움 | 충돌과 정보 노출 위험 |
| UUID 기반 Key | 충돌과 추측 가능성 감소 | 원본 파일명 별도 저장 |

Bucket은 보안 경계, 데이터 소유권, 수명 주기와 운영 책임이 다를 때 분리한다.

---

# AWS CLI에서는

서울 리전에 Bucket을 생성하고 파일을 업로드한다.

```bash
aws s3api create-bucket \
  --bucket my-service-123456789012 \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2

aws s3api put-object \
  --bucket my-service-123456789012 \
  --key users/42/profile/sample.png \
  --body ./sample.png \
  --content-type image/png
```

prefix로 객체를 조회하고 메타데이터를 확인한다.

```bash
aws s3api list-objects-v2 \
  --bucket my-service-123456789012 \
  --prefix users/42/profile/

aws s3api head-object \
  --bucket my-service-123456789012 \
  --key users/42/profile/sample.png
```

---

# Spring Boot에서는 어떻게 쓰는가

Bucket 이름과 리전은 환경별 설정으로 분리한다.

```yaml
aws:
  region: ap-northeast-2
  s3:
    bucket: my-service-prod-123456789012
```

파일명을 직접 Key로 사용하지 않고 서버가 고유 Key를 생성한다.

```java
@Service
public class ProfileImageStorage {

    private final S3Client s3Client;
    private final String bucket;

    public ProfileImageStorage(
            S3Client s3Client,
            @Value("${aws.s3.bucket}") String bucket) {
        this.s3Client = s3Client;
        this.bucket = bucket;
    }

    public String upload(long userId, String extension, byte[] content) {
        String key = "users/%d/profile/%s.%s"
                .formatted(userId, UUID.randomUUID(), extension);

        PutObjectRequest request = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType("image/" + extension)
                .metadata(Map.of("owner-id", Long.toString(userId)))
                .build();

        s3Client.putObject(request, RequestBody.fromBytes(content));
        return key;
    }
}
```

사용자가 보낸 확장자와 `Content-Type`은 위조될 수 있으므로 허용 목록과 파일 시그니처를 함께 검사한다.

---

# 실무에서는 어떻게 사용할까

Key는 `도메인/소유자/용도/UUID.확장자`처럼 정책 적용에 필요한 안정적인 prefix를 갖도록 설계한다.

DB에는 `bucket`, `key`, `originalFilename`, `contentType`, `size`, `ownerId`와 업로드 상태를 저장한다.

객체 태그는 Lifecycle이나 접근 정책 조건에 사용할 수 있지만 핵심 업무 조회를 S3 태그 검색에 의존하지 않는다.

삭제는 DB와 S3의 분산 작업이므로 삭제 상태를 기록하고 재시도 가능한 비동기 정리를 고려한다.

---

# 장애 사례와 주의할 점

같은 Key에 대한 `PutObject`는 파일 추가가 아니라 객체 교체이므로 이름 충돌이 데이터 유실로 이어진다.

전체 URL을 DB의 유일한 값으로 두면 Bucket 또는 CloudFront 도메인 변경 시 대규모 수정이 필요하다.

Bucket 정책에서 모든 Principal을 허용하면 공개 노출이 발생할 수 있으므로 Block Public Access를 유지한다.

IAM Role에는 필요한 Bucket과 prefix의 `GetObject`, `PutObject` 같은 최소 권한만 부여한다.

---

# 비용과 성능 고려사항

객체 저장 용량, PUT·GET·LIST 요청 수, 데이터 전송과 Storage Class가 비용에 영향을 준다.

목록 조회로 업무 데이터를 찾기보다 DB에서 Key를 조회해 직접 접근하는 구조가 예측 가능하다.

작은 객체를 지나치게 많이 만들면 요청 수와 객체 관리 비용이 커질 수 있다.

공개 다운로드는 CloudFront 캐시를 사용하면 S3 원본 요청과 아웃바운드 전송을 줄일 수 있다.

---

# 기억해야 할 내용

- Bucket은 객체를 담는 최상위 경계이고 Object는 실제 저장 단위이다.
- 객체는 Bucket과 Key의 조합으로 식별한다.
- Key의 `/`는 실제 디렉터리가 아니라 prefix 표현이다.
- 같은 Key로 쓰면 기존 객체가 덮어써지므로 고유 Key를 생성한다.
- 원본 파일명과 업무 메타데이터는 DB에 별도로 보관한다.
- Bucket은 기본적으로 비공개로 두고 IAM 최소 권한을 적용한다.

---

# 다음 Chapter

다음 Chapter에서는 동일한 Key의 변경 이력을 보존하는 **Versioning**을 학습한다.

실수로 객체를 덮어쓰거나 삭제했을 때 이전 버전을 복구하는 원리를 살펴본다.


