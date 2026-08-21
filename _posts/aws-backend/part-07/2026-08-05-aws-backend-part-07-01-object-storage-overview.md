---
title: "Chapter 01. Object Storage Overview"
permalink: /aws-backend/part-07/01-object-storage-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Object Storage Overview
## 확장 가능한 파일 서비스의 출발점

> **학습 목표**
>
> - Object Storage의 개념을 설명할 수 있다.
> - 블록, 파일, 객체 스토리지의 차이를 설명할 수 있다.
> - 파일 업로드와 다운로드 흐름을 설명할 수 있다.
> - Spring Boot에서 AWS SDK for Java v2로 S3를 사용할 수 있다.

---

# 왜 Object Storage가 필요한가

쇼핑몰 서버가 사용자 프로필 이미지와 상품 이미지를 자신의 로컬 디스크에 저장한다고 가정한다.

서버가 한 대일 때는 문제가 없어 보이지만 트래픽 증가로 서버를 세 대로 늘리면 어느 서버에 파일이 저장되었는지에 따라 조회 결과가 달라진다.

컨테이너가 교체되거나 인스턴스가 종료되면 로컬 디스크의 파일을 잃을 수도 있다.

공유 파일 시스템을 붙이는 방법도 있지만 용량 계획, 장애 복구, 확장과 운영을 직접 책임져야 한다.

애플리케이션의 실행 환경과 파일 저장소를 분리하면 모든 서버가 같은 파일을 바라보고 서버를 자유롭게 확장하거나 교체할 수 있다.

AWS에서는 이 독립적인 파일 저장소 역할을 **Amazon S3(Simple Storage Service)** 가 담당한다.

---

# Object Storage란?

Object Storage는 데이터를 계층형 파일 시스템의 파일이 아니라 고유한 키를 가진 **객체(Object)** 로 저장하는 방식이다.

객체는 데이터와 메타데이터, 키를 가진다.

S3는 AWS의 대표적인 관리형 Object Storage 서비스이다.

S3는 VPC 내부에 배치하는 서버가 아니라 **VPC 밖에서 동작하는 리전 서비스**이며, 인터넷 또는 VPC Endpoint를 통해 접근한다.

S3는 객체의 생성, 덮어쓰기, 삭제와 목록 조회에 대해 강한 일관성(**strong read-after-write consistency**)을 제공한다.

![S3 file service](/assets/aws-backend/s3-file-service.png)

---

# 저장 방식 비교

| 구분 | 블록 스토리지 | 파일 스토리지 | 객체 스토리지 |
|------|---------------|---------------|---------------|
| 접근 단위 | 블록 | 파일과 디렉터리 | 키를 가진 객체 |
| 대표 서비스 | EBS | EFS | S3 |
| 접근 방식 | 운영체제에 디스크 연결 | NFS 같은 파일 프로토콜 | HTTP 기반 API |
| 수정 방식 | 일부 블록 수정 가능 | 파일 일부 수정 가능 | 객체 전체 교체 |
| 적합한 용도 | DB, 부트 볼륨 | 공유 파일 시스템 | 이미지, 동영상, 백업 |
| 확장 책임 | 볼륨 크기 관리 | 파일 시스템 관리 | 서비스가 확장 관리 |

S3는 일반 디렉터리 구조 대신 Bucket 안에서 Object Key로 객체를 식별한다.

애플리케이션 서버에 파일을 직접 저장하지 않으므로 서버 Scale Out에 유리하다.

---

# 동작 흐름

일반적인 파일 서비스는 애플리케이션이 메타데이터를 관리하고 S3가 실제 바이트 데이터를 보관하는 구조이다.

```
Browser
   │  업로드 요청
   ▼
Spring Boot ───── 메타데이터 ───▶ Database
   │
   │ PutObject
   ▼
  S3 Bucket
   │
   │ GetObject 또는 CloudFront
   ▼
Browser
```

1. 브라우저가 파일과 업무 정보를 애플리케이션에 전달한다.
2. 애플리케이션은 인증, 확장자, 크기와 권한을 검증한다.
3. 애플리케이션은 고유한 Object Key를 만들고 S3에 객체를 저장한다.
4. DB에는 파일 바이트가 아니라 Bucket, Key, 원본 파일명과 소유자 같은 메타데이터를 저장한다.
5. 조회 시 권한을 확인한 뒤 S3 또는 CloudFront를 통해 파일을 제공한다.

대용량 파일은 애플리케이션을 경유하지 않고 [Presigned URL](/aws-backend/part-07/05-presigned-url/)로 브라우저가 S3에 직접 업로드하도록 설계할 수 있다.

---

# AWS CLI에서는

먼저 현재 자격 증명으로 접근 가능한 Bucket을 확인한다.

```bash
aws s3 ls
```

로컬 파일을 S3에 업로드하고 다시 내려받는 대표 명령은 다음과 같다.

```bash
aws s3 cp ./profile.png s3://my-service-bucket/profiles/profile.png
aws s3 cp s3://my-service-bucket/profiles/profile.png ./downloaded.png
```

`s3://버킷/키`는 URI 표현이며 객체 자체는 HTTP API로 관리된다.

---

# Spring Boot에서는 어떻게 쓰는가

AWS SDK for Java v2의 S3 모듈을 의존성에 추가한다.

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
</dependency>
```

리전과 Bucket 이름은 코드에 하드코딩하지 않고 설정으로 분리한다.

```yaml
aws:
  region: ap-northeast-2
  s3:
    bucket: my-service-bucket
```

애플리케이션에서는 기본 자격 증명 공급자 체인을 활용하는 `S3Client`를 Bean으로 등록한다.

```java
@Configuration
public class S3Configuration {
    @Bean
    S3Client s3Client(@Value("${aws.region}") String region) {
        return S3Client.builder().region(Region.of(region)).build();
    }
}
```

운영 환경에서는 ECS Task Role이나 EC2 Instance Profile을 사용하고 Access Key를 설정 파일에 저장하지 않는다.

```java
@Service
public class ObjectStorageService {
    private final S3Client s3Client;
    private final String bucket;
    public ObjectStorageService(
            S3Client s3Client,
            @Value("${aws.s3.bucket}") String bucket) {
        this.s3Client = s3Client;
        this.bucket = bucket;
    }
    public void upload(String key, byte[] content, String contentType) {
        PutObjectRequest request = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(contentType)
                .build();
        s3Client.putObject(request, RequestBody.fromBytes(content));
    }
}
```

큰 파일을 `byte[]`로 모두 읽으면 Heap 사용량이 커지므로 실제 서비스에서는 스트리밍, Presigned URL 또는 Multipart Upload를 선택한다.

---

# Part 7 학습 지도

| Chapter | 핵심 질문 |
|---------|-----------|
| [02. Bucket and Object](/aws-backend/part-07/02-bucket-object/) | Bucket과 Key를 어떻게 설계하는가 |
| [03. Versioning](/aws-backend/part-07/03-versioning/) | 덮어쓰기와 삭제를 어떻게 복구하는가 |
| [04. Lifecycle](/aws-backend/part-07/04-lifecycle/) | 오래된 데이터를 어떻게 자동 정리하는가 |
| [05. Presigned URL](/aws-backend/part-07/05-presigned-url/) | 클라이언트에 임시 권한을 어떻게 위임하는가 |
| [06. Multipart Upload](/aws-backend/part-07/06-multipart-upload/) | 대용량 파일 실패 범위를 어떻게 줄이는가 |
| [07. CloudFront](/aws-backend/part-07/07-cloudfront/) | 파일을 전 세계에 어떻게 빠르게 제공하는가 |

---

# 실무에서는 어떻게 사용할까

DB에는 파일 전체 URL보다 논리적인 Object Key와 파일 소유 정보를 저장하는 편이 도메인 변경과 CDN 교체에 유연하다.

원본 이미지는 S3에 비공개로 저장하고 리사이징된 파생 이미지를 별도 prefix에 생성하면 원본 보존과 캐시 정책을 분리할 수 있다.

사용자 입력 파일은 실행 가능한 콘텐츠일 수 있으므로 업로드 Bucket과 정적 웹 자산 Bucket을 분리하고 콘텐츠 타입을 신뢰하지 않는다.

---

# 장애 사례와 주의할 점

여러 사용자가 같은 원본 파일명을 Key로 사용해 `profile.png`를 계속 덮어쓰는 장애가 발생할 수 있다.

Key에는 사용자 ID, 업무 ID 또는 UUID를 포함하고 원본 파일명은 별도 메타데이터로 보관해야 한다.

Bucket을 공개해 다운로드 문제를 해결하면 개인정보와 내부 파일이 외부에 노출될 수 있다.

Block Public Access를 유지하고 애플리케이션 권한, Presigned URL 또는 CloudFront OAC로 필요한 접근만 허용한다.

S3 호출 실패를 무조건 재시도하면 요청 폭주가 커질 수 있으므로 SDK 재시도 정책과 애플리케이션의 멱등성을 함께 검토한다.

---

# 비용과 성능 고려사항

S3 비용은 저장 용량뿐 아니라 요청 수, 데이터 검색, 스토리지 클래스 전환과 외부 데이터 전송에 의해 달라진다.

작은 객체가 매우 많으면 총 용량이 작더라도 요청과 객체 관리 비용의 비중이 커질 수 있다.

접근 빈도와 복구 시간 요구가 다른 파일을 같은 스토리지 클래스에 두지 말고 Lifecycle로 구분한다.

공개 콘텐츠 앞에 CloudFront를 두면 Edge Cache가 반복 요청을 처리하여 S3 요청과 S3에서 나가는 데이터 전송을 줄일 수 있다.

---

# 기억해야 할 내용

- S3는 서버 디스크가 아니라 독립적으로 확장되는 Object Storage이다.
- 객체는 Key, 데이터와 메타데이터로 구성된다.
- S3는 VPC 밖의 리전 서비스이며 강한 읽기 후 쓰기 일관성을 제공한다.
- 애플리케이션 서버와 파일 저장소를 분리하면 Scale Out과 서버 교체가 쉬워진다.
- DB에는 보통 파일 바이트가 아니라 Object Key와 업무 메타데이터를 저장한다.
- Bucket 공개 대신 최소 권한과 안전한 전달 경로를 사용한다.

---

# 다음 Chapter

다음 Chapter에서는 S3의 핵심 구성 단위인 **Bucket과 Object**를 학습한다.

Bucket의 경계와 Object Key 설계가 권한, 운영과 덮어쓰기 방지에 어떤 영향을 주는지 살펴본다.


