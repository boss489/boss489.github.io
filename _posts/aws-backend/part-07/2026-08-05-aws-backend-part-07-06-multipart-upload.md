---
title: "Chapter 06. Multipart Upload"
permalink: /aws-backend/part-07/06-multipart-upload/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Multipart Upload
## 대용량 전송의 실패 범위를 줄이는 방법

> **학습 목표**
>
> - Multipart Upload의 목적을 설명할 수 있다.
> - 생성, 파트 업로드와 완료 흐름을 이해한다.
> - 큰 파일 업로드 실패 시 재시도 방식을 설명할 수 있다.
> - 미완료 업로드를 안전하게 정리할 수 있다.

---

# 왜 Multipart Upload가 필요한가

사용자가 수 GB 동영상을 하나의 요청으로 업로드하다가 마지막 구간에서 네트워크가 끊겼다고 가정한다.

단일 업로드는 처음부터 전체 파일을 다시 전송해야 하므로 시간과 대역폭을 낭비한다.

긴 연결 하나에 의존하면 모바일 네트워크 변화와 일시적인 장애에 취약하다.

파일을 여러 파트로 나누면 이미 성공한 파트는 유지하고 실패한 파트만 다시 보낼 수 있다.

서로 독립적인 파트를 병렬로 전송하면 충분한 네트워크 환경에서 전체 업로드 시간도 줄일 수 있다.

---

# Multipart Upload란?

Multipart Upload는 하나의 객체를 여러 파트로 나누어 업로드한 뒤 S3가 최종 객체로 조립하는 방식이다.

업로드 생성 시 `uploadId`가 발급되고 각 파트는 `partNumber`와 응답의 `ETag`로 추적한다.

완료 요청에는 성공한 모든 파트의 번호와 ETag를 순서대로 전달해야 한다.

파트 업로드가 끝났더라도 완료 요청 전에는 정상 객체로 조회할 수 없다.

Multipart로 완성된 객체의 ETag를 파일 전체의 단순 MD5로 가정해서는 안 된다.

---

# 동작 흐름

```
Client ── CreateMultipartUpload ──▶ S3
Client ◀────── uploadId ─────────── S3
   │
   ├── UploadPart #1 ─────────────▶ S3
   ├── UploadPart #2 ─────────────▶ S3
   └── UploadPart #3 ─────────────▶ S3
   │
   └── Complete(parts + ETags) ───▶ S3
Client ◀──── 완성된 Object ──────── S3
```

1. 클라이언트가 Bucket과 Key로 Multipart Upload를 생성한다.
2. 파일을 적절한 크기의 파트로 나누고 각 파트에 번호를 부여한다.
3. 파트를 순차 또는 병렬로 업로드하고 ETag를 기록한다.
4. 실패한 파트만 동일한 번호로 재시도한다.
5. 모든 파트의 번호와 ETag를 전달해 완료 요청을 보낸다.
6. 더 진행할 수 없으면 Abort 요청으로 업로드된 파트를 삭제한다.

S3에서 파트는 마지막 파트를 제외하면 최소 5MiB이어야 하며 하나의 Multipart Upload는 최대 10,000개 파트를 지원한다.

---

# 업로드 방식 비교

| 구분 | 단일 PutObject | Multipart Upload |
|------|-----------------|------------------|
| 요청 구조 | 객체 한 번 전송 | 여러 파트와 완료 요청 |
| 실패 재시도 | 전체 재전송 | 실패 파트만 재전송 |
| 병렬 처리 | 어려움 | 가능 |
| 상태 관리 | 단순 | uploadId, ETag 관리 |
| 중단 정리 | 별도 파트 없음 | Abort 또는 Lifecycle 필요 |
| 적합한 대상 | 작은 파일 | 대용량 파일과 불안정한 네트워크 |

파트를 너무 작게 나누면 요청 수가 늘고 너무 크게 나누면 재시도 범위와 병렬성 이점이 줄어든다.

---

# AWS CLI에서는

AWS CLI의 고수준 명령은 큰 파일에서 Multipart Upload를 자동으로 사용할 수 있다.

```bash
aws s3 cp ./large-video.mp4 \
  s3://my-service-bucket/videos/large-video.mp4
```

진행 중인 업로드를 조회하고 더 이상 필요 없는 업로드를 중단한다.

```bash
aws s3api list-multipart-uploads \
  --bucket my-service-bucket

aws s3api abort-multipart-upload \
  --bucket my-service-bucket \
  --key videos/large-video.mp4 \
  --upload-id UPLOAD_ID
```

운영에서는 Lifecycle의 `AbortIncompleteMultipartUpload` 규칙으로 수동 정리 누락을 보완한다.

---

# Spring Boot에서는 어떻게 쓰는가

SDK v2의 고수준 `S3TransferManager`는 파트 분할, 병렬 전송과 재시도를 단순화한다.

비동기 S3 클라이언트와 Transfer Manager를 Bean으로 등록한다.

```java
@Configuration
public class S3TransferConfiguration {

    @Bean
    S3AsyncClient s3AsyncClient(@Value("${aws.region}") String region) {
        return S3AsyncClient.crtBuilder()
                .region(Region.of(region))
                .targetThroughputInGbps(10.0)
                .minimumPartSizeInBytes(8L * 1024 * 1024)
                .build();
    }

    @Bean
    S3TransferManager s3TransferManager(S3AsyncClient client) {
        return S3TransferManager.builder()
                .s3Client(client)
                .build();
    }
}
```

서비스는 파일 경로를 Transfer Manager에 전달하고 완료를 기다린다.

```java
@Service
public class LargeObjectUploadService {

    private final S3TransferManager transferManager;
    private final String bucket;

    public LargeObjectUploadService(
            S3TransferManager transferManager,
            @Value("${aws.s3.bucket}") String bucket) {
        this.transferManager = transferManager;
        this.bucket = bucket;
    }

    public CompletedFileUpload upload(Path source, String key) {
        UploadFileRequest request = UploadFileRequest.builder()
                .source(source)
                .putObjectRequest(put -> put.bucket(bucket).key(key))
                .build();

        return transferManager.uploadFile(request)
                .completionFuture()
                .join();
    }
}
```

CRT 기반 클라이언트를 사용하려면 프로젝트에 AWS SDK v2의 S3 Transfer Manager와 CRT 관련 의존성을 추가해야 한다.

---

# 실무에서는 어떻게 사용할까

서버가 파일을 직접 받지 않는 구조에서는 각 파트의 Presigned URL을 발급하고 클라이언트가 S3로 직접 업로드하게 할 수 있다.

DB나 Redis에 `uploadId`, Key, 파트 번호, ETag와 상태를 저장하면 중단 후 재개 흐름을 만들 수 있다.

완료 요청은 중복 실행될 수 있으므로 완료 상태를 멱등하게 처리하고 최종 객체를 `HeadObject`로 확인한다.

사용자가 취소하면 즉시 Abort를 호출하고 예외나 브라우저 종료로 누락된 작업은 Lifecycle로 정리한다.

---

# 장애 사례와 주의할 점

파트 업로드 후 Complete나 Abort를 호출하지 않으면 조각이 계속 저장되어 비용이 발생한다.

클라이언트가 ETag 목록을 잃으면 완료 요청을 구성하기 어려우므로 서버 또는 지속 가능한 클라이언트 저장소에 상태를 남긴다.

서로 다른 업로드의 `uploadId`나 ETag를 섞으면 완료가 실패하므로 Key와 사용자 소유권을 함께 검증한다.

무제한 병렬 전송은 네트워크, 파일 디스크립터와 메모리를 고갈시킬 수 있으므로 동시성을 제한한다.

---

# 비용과 성능 고려사항

각 파트는 별도 업로드 요청이므로 파트가 많을수록 요청 수가 증가한다.

병렬성은 대역폭을 높일 수 있지만 클라이언트와 네트워크 한계를 넘으면 오히려 재시도와 지연이 늘어난다.

미완료 파트도 저장 용량을 사용하므로 정리 기간을 업무상 재개 가능 시간과 균형 있게 정한다.

CloudFront는 일반적인 업로드 대상이 아니라 다운로드 캐시 계층이며 업로드는 S3 엔드포인트를 사용한다.

---

# 기억해야 할 내용

- Multipart Upload는 큰 객체를 여러 파트로 나누어 전송한다.
- 실패한 파트만 재시도하여 전체 재전송을 피한다.
- uploadId, partNumber와 ETag를 정확히 관리한다.
- 모든 파트 업로드 후 Complete 요청이 필요하다.
- 취소 시 Abort를 호출하고 Lifecycle 정리를 함께 둔다.
- 파트 크기와 병렬성은 요청 수와 자원 사용을 고려해 결정한다.

---

# 다음 Chapter

다음 Chapter에서는 S3 객체를 사용자 가까운 Edge에서 제공하는 **CloudFront**를 학습한다.

캐시로 응답 시간을 줄이고 OAC로 비공개 S3 오리진을 보호하는 방법을 살펴본다.


