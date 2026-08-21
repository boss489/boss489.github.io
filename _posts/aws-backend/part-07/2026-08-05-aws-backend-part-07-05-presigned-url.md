---
title: "Chapter 05. Presigned URL"
permalink: /aws-backend/part-07/05-presigned-url/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Presigned URL
## 클라이언트에 제한된 S3 권한 위임하기

> **학습 목표**
>
> - Presigned URL의 목적을 설명할 수 있다.
> - 서버를 거치지 않는 직접 업로드 흐름을 이해한다.
> - 만료 시간과 작업 범위를 제한할 수 있다.
> - AWS SDK for Java v2로 URL을 발급할 수 있다.

---

# 왜 Presigned URL이 필요한가

사용자가 500MB 동영상을 Spring Boot API에 보내고 서버가 다시 S3에 업로드한다고 가정한다.

같은 데이터가 브라우저에서 서버로, 서버에서 S3로 두 번 전송되고 애플리케이션의 연결, 메모리와 네트워크를 오래 점유한다.

동시 업로드가 늘면 핵심 API 요청까지 느려지고 서버를 확장해야 하는 원인이 된다.

브라우저에 AWS Access Key를 전달하는 것은 장기 자격 증명을 노출하므로 허용할 수 없다.

서버가 인증과 정책을 판단한 뒤 특정 작업만 짧게 허용하는 Presigned URL을 발급하면 클라이언트가 S3와 직접 통신할 수 있다.

---

# Presigned URL이란?

Presigned URL은 AWS 자격 증명으로 서명되어 제한된 시간 동안 특정 S3 요청을 수행할 수 있는 URL이다.

URL에는 대상 Bucket, Key, HTTP 메서드, 만료 정보와 서명이 반영된다.

URL을 받은 주체는 별도 AWS 자격 증명 없이 서명된 범위 안에서 요청할 수 있다.

Presigned URL을 사용해도 Bucket이 공개로 바뀌는 것은 아니며 URL을 생성한 IAM 주체의 권한을 넘어설 수 없다.

![S3 file service](/assets/aws-backend/s3-file-service.png)

---

# 동작 흐름

업로드 완료를 URL 발급과 분리하고 서버가 결과를 검증하는 것이 핵심이다.

```
Browser ── 1. 발급 요청 ──▶ Spring Boot
Browser ◀─ 2. URL + Key ─── Spring Boot
   │
   └──── 3. PUT file ─────▶ S3
Browser ── 4. 완료 통지 ──▶ Spring Boot
                              │
                              └─ 5. HeadObject 후 DB 확정
```

1. 브라우저가 파일명, 크기와 콘텐츠 타입을 서버에 전달한다.
2. 서버가 사용자 권한을 검증하고 고유 Key로 Presigned PUT URL을 발급한다.
3. 브라우저가 정해진 메서드와 헤더로 S3에 직접 업로드한다.
4. 브라우저가 업로드 완료와 Key를 서버에 알린다.
5. 서버가 `HeadObject`로 실제 객체 크기와 메타데이터를 검증한 뒤 DB 상태를 완료로 바꾼다.

URL 발급 성공은 업로드 성공이 아니므로 `PENDING`, `COMPLETED`, `EXPIRED` 같은 상태 모델이 필요하다.

---

# 업로드 방식 비교

| 구분 | 서버 경유 업로드 | Presigned URL 직접 업로드 |
|------|------------------|----------------------------|
| 데이터 경로 | Browser → App → S3 | Browser → S3 |
| 서버 부하 | 연결과 대역폭 사용 | 발급과 검증만 수행 |
| 검증 시점 | 저장 전 스트림 검증 가능 | 발급 전·완료 후 검증 |
| 자격 증명 | 서버만 보유 | 서버만 보유 |
| 대용량 확장성 | 서버 자원에 영향 | S3로 부하 분산 |
| 구현 복잡도 | 단순 | 상태와 CORS 설계 필요 |

다운로드에도 Presigned GET URL을 사용할 수 있지만 대규모 공개 콘텐츠는 CloudFront가 더 적합하다.

---

# AWS CLI에서는

CLI로 제한된 시간의 다운로드 URL을 생성할 수 있다.

```bash
aws s3 presign \
  s3://my-service-bucket/private/report.pdf \
  --expires-in 300
```

생성된 PUT URL은 서명에 포함된 헤더를 동일하게 보내야 한다.

```bash
curl -X PUT \
  -H "Content-Type: image/png" \
  --upload-file ./profile.png \
  "PRESIGNED_URL"
```

브라우저 직접 업로드에는 Bucket CORS에서 서비스 Origin, 메서드와 헤더를 필요한 범위로 허용해야 한다.

---

# Spring Boot에서는 어떻게 쓰는가

SDK v2의 `S3Presigner`를 Bean으로 등록하고 종료 시 자원을 닫는다.

```java
@Configuration
public class S3PresignerConfiguration {

    @Bean
    S3Presigner s3Presigner(@Value("${aws.region}") String region) {
        return S3Presigner.builder()
                .region(Region.of(region))
                .build();
    }
}
```

서비스는 서버가 생성한 Key와 짧은 만료 시간을 사용한다.

```java
@Service
public class PresignedUploadService {

    private final S3Presigner presigner;
    private final String bucket;

    public PresignedUploadService(
            S3Presigner presigner,
            @Value("${aws.s3.bucket}") String bucket) {
        this.presigner = presigner;
        this.bucket = bucket;
    }

    public PresignedUpload issue(long userId, String contentType) {
        String key = "uploads/%d/%s".formatted(userId, UUID.randomUUID());

        PutObjectRequest put = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(contentType)
                .build();

        PresignedPutObjectRequest signed = presigner.presignPutObject(request -> request
                .signatureDuration(Duration.ofMinutes(5))
                .putObjectRequest(put));

        return new PresignedUpload(key, signed.url().toString());
    }
}

public record PresignedUpload(String key, String url) {
}
```

`Content-Type`을 서명했다면 클라이언트는 업로드할 때 정확히 같은 헤더를 보내야 한다.

---

# 실무에서는 어떻게 사용할까

URL 발급 전에 사용자별 업로드 권한, 허용 타입, 예상 크기와 할당량을 검증한다.

Key는 서버가 생성하고 사용자 ID나 업무 ID 아래에 UUID를 사용하여 다른 사용자의 경로를 선택하지 못하게 한다.

완료 통지 후 `HeadObject` 결과를 검증하고 바이러스 검사나 이미지 변환을 비동기 파이프라인으로 연결한다.

미완료 상태는 Lifecycle로 삭제하고 DB의 발급 기록도 만료 처리한다.

---

# 장애 사례와 주의할 점

Presigned URL은 만료 전까지 URL 자체가 임시 자격 증명처럼 동작하므로 로그, 채팅과 분석 도구에 노출하지 않는다.

만료 시간을 길게 두면 유출 시 악용 가능한 시간이 늘어나고 너무 짧으면 느린 네트워크에서 업로드 시작 전에 만료될 수 있다.

PUT URL은 일반적으로 콘텐츠 길이 자체를 강제하는 수단이 제한적이므로 완료 후 크기를 검증하고 필요하면 Presigned POST 정책을 검토한다.

CORS를 `*`로 열기보다 실제 프론트엔드 Origin과 필요한 메서드만 허용한다.

---

# 비용과 성능 고려사항

직접 업로드는 애플리케이션 서버의 네트워크 경유와 장시간 연결을 제거하지만 S3 PUT 요청과 저장 비용은 그대로 발생한다.

사용자가 URL만 발급받고 업로드하지 않으면 DB에 유령 상태가 쌓이므로 만료 정리가 필요하다.

다운로드를 매번 Presigned S3 URL로 제공하면 S3 요청과 아웃바운드 전송이 발생하므로 반복 콘텐츠는 CloudFront를 고려한다.

대용량 파일은 [Multipart Upload](/aws-backend/part-07/06-multipart-upload/)와 결합하여 실패한 조각만 재시도할 수 있다.

---

# 기억해야 할 내용

- Presigned URL은 특정 S3 작업을 제한된 시간 동안 위임한다.
- Bucket을 공개하지 않고 클라이언트 직접 업로드를 구현할 수 있다.
- 서버가 Key를 생성하고 발급 전 권한과 정책을 검증한다.
- URL 발급과 업로드 완료는 서로 다른 상태이다.
- 완료 후 객체 크기와 메타데이터를 다시 검증한다.
- URL 유출, 과도한 만료 시간과 넓은 CORS를 피한다.

---

# 다음 Chapter

다음 Chapter에서는 큰 파일을 여러 조각으로 나누는 **Multipart Upload**를 학습한다.

네트워크 오류가 발생했을 때 전체 파일 대신 실패한 조각만 재시도하는 흐름을 살펴본다.


