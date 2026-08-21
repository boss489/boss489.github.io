---
title: "Chapter 08. File Delivery Architecture"
permalink: /aws-backend/part-16/08-file-delivery-architecture/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. File Delivery Architecture
## 업로드와 변환과 전송을 분리한 파일 파이프라인

> **학습 목표**
>
> - Presigned URL로 대용량 파일을 직접 업로드한다.
> - S3 Event 기반 후처리와 악성 파일 검사를 설계한다.
> - CloudFront OAC로 원본 Bucket을 비공개로 유지한다.
> - 불변 Key와 Versioning으로 캐시와 복구를 단순화한다.

---

# 요구사항과 실패 시나리오

브라우저는 인증된 API에서 서버가 생성한 Object Key와 짧은 Presigned URL을 받는다.

파일 본문은 ECS를 통과하지 않고 S3 임시 Prefix로 직접 업로드한다.

URL 발급 성공과 업로드 완료는 다르므로 PENDING과 VERIFIED 상태를 분리한다.

완료 이벤트는 SQS로 버퍼링하여 이미지 변환 Consumer의 장애를 흡수한다.

업로드 크기와 Content Type은 발급 전과 HeadObject 후에 모두 검증한다.

피크 시간에 하위 시스템 하나가 느려졌을 때 전체 주문 경로가 함께 멈추는 구조인지 먼저 질문한다.

네트워크 재시도와 중복 메시지와 AZ 장애를 정상적인 운영 조건으로 간주하고 설계를 검증한다.

---

# 설계 원칙

요구사항을 측정 가능한 지표로 바꾸고 제약과 가정을 명시한다.

동기 경로는 사용자 응답에 꼭 필요한 작업으로 줄이고 나머지는 비동기로 분리한다.

상태는 명시적인 원본 저장소에 두고 캐시와 실행 노드는 교체 가능하게 만든다.

실패를 숨기기보다 Timeout과 격리와 재처리와 관측 지점을 설계한다.

가장 복잡한 기술보다 현재 규모에서 운영할 수 있고 진화 가능한 결정을 선택한다.

---

# 아키텍처

```text
Browser -- request URL --> ECS API
Browser <-- signed PUT ---- ECS API
   |
   +---- upload ----> S3 quarantine
                         | Event
                         v
                      SQS queue
                         |
                Scan/Transform Worker
                         | verified immutable key
CloudFront -- OAC --> S3 private origin --> Browser
```

다이어그램의 화살표는 실제 데이터 경로를 나타내며 관측 도구는 업무 저장소 뒤에 직렬로 놓인 구성 요소가 아니다.

---

# 선택지 비교

| 기준 | 선택 A | 선택 B | 결정 기준 |
|---|---|---|---|
| 방식 | 서버 경유 | 직접 업로드 | 결정 |
| 대역폭 | App 두 번 전송 | Browser에서 S3 | 대용량은 직접 |
| 검사 | 스트림 선검사 | 비동기 후검사 | 격리 상태 필요 |
| 권한 | 서버만 S3 접근 | 서명 범위 접근 | 짧은 만료 |
| 전송 | App 응답 | CloudFront Edge | 반복 다운로드 |

비교표의 결정은 영구 결론이 아니라 현재 요구와 팀의 운영 역량에 기반한 선택이다.

---

# 동작 흐름과 결정

1. 브라우저는 인증된 API에서 서버가 생성한 Object Key와 짧은 Presigned URL을 받는다.

2. 파일 본문은 ECS를 통과하지 않고 S3 임시 Prefix로 직접 업로드한다.

3. URL 발급 성공과 업로드 완료는 다르므로 PENDING과 VERIFIED 상태를 분리한다.

4. 완료 이벤트는 SQS로 버퍼링하여 이미지 변환 Consumer의 장애를 흡수한다.

5. 업로드 크기와 Content Type은 발급 전과 HeadObject 후에 모두 검증한다.

6. 신뢰하지 않는 파일은 격리 Bucket 또는 Prefix에서 악성 코드 검사 후 공개 가능 상태로 이동한다.

7. S3 Event는 중복되거나 순서가 바뀔 수 있으므로 Bucket, Key, Version ID를 멱등 키로 사용한다.

8. CloudFront는 OAC로 Private S3 Origin에 접근하고 사용자는 S3 URL에 직접 접근하지 않는다.

9. 파일 교체는 같은 Key 덮어쓰기보다 콘텐츠 해시나 UUID가 포함된 불변 Key를 생성한다.

10. Versioning과 Lifecycle은 복구 가능성과 이전 버전 저장 비용 사이에서 보존 기간을 정한다.

결정 기록에는 선택하지 않은 대안과 다시 검토할 조건도 함께 남긴다.

---

# 구현 예

```java
PutObjectRequest put = PutObjectRequest.builder()
        .bucket(uploadBucket)
        .key("quarantine/" + UUID.randomUUID())
        .contentType(request.contentType())
        .build();
PresignedPutObjectRequest signed = presigner.presignPutObject(builder -> builder
        .signatureDuration(Duration.ofMinutes(5))
        .putObjectRequest(put));
```

예제 설정과 수치는 동작 원리를 보여 주기 위한 가정이며 운영값은 측정과 부하 테스트로 확정한다.

---

# 실무 Trade-off

가용성을 높이는 복제와 대기 자원은 비용을 늘리므로 업무 중요도별로 등급을 나눈다.

강한 정합성은 안전한 결정을 돕지만 지연과 결합도를 높일 수 있어 필요한 경계에만 적용한다.

관리형 서비스는 운영 부담을 줄이지만 서비스 한도와 종속성과 비용 모델을 이해해야 한다.

비동기화는 장애 격리와 흡수력을 높이지만 상태 추적과 중복 처리와 재처리 운영이 필요하다.

초기 설계는 단순하게 시작하되 지표와 인터페이스를 남겨 병목이 확인되면 교체할 수 있게 한다.

---

# 장애, 보안과 비용

장애 시나리오는 증상, 탐지 지표, 자동 완화, 수동 Runbook과 복구 확인 순서로 작성한다.

IAM Role에는 필요한 Action과 Resource만 허용하고 장기 Access Key를 애플리케이션에 저장하지 않는다.

전송 구간 TLS와 저장 데이터 KMS 암호화를 적용하고 개인정보 접근은 감사 가능하게 남긴다.

재시도는 지수 Backoff와 Jitter와 횟수 제한을 사용하여 장애 중 부하 증폭을 막는다.

비용은 컴퓨팅 시간, 저장량, 요청 수, 데이터 전송과 고정 네트워크 비용으로 나눠 추적한다.

특정 달러 가격은 Region과 시점과 약정에 따라 달라지므로 확정값 대신 AWS Pricing Calculator와 실제 청구로 검증한다.

---

# 검증 질문

- 이 결정이 해결하려는 구체적인 요구사항과 제약은 무엇인가.
- 한 AZ 또는 외부 의존성이 실패하면 사용자가 어떤 응답을 받는가.
- 중복 요청과 지연된 이벤트와 부분 성공을 어떻게 식별하고 복구하는가.
- 가장 먼저 포화되는 자원과 이를 알려 주는 선행 지표는 무엇인가.
- 보안 경계와 데이터 소유자와 최소 권한의 증거는 무엇인가.
- 예상 비용의 가장 큰 항목과 트래픽 증가에 따른 기울기는 무엇인가.
- 설계 가정을 어떤 부하 테스트와 장애 훈련으로 반증할 수 있는가.

---

# 기억해야 할 내용

- Presigned URL로 대용량 파일을 직접 업로드한다.
- S3 Event 기반 후처리와 악성 파일 검사를 설계한다.
- CloudFront OAC로 원본 Bucket을 비공개로 유지한다.
- 불변 Key와 Versioning으로 캐시와 복구를 단순화한다.
- 요구사항에서 제약과 대안을 거쳐 결정을 내리고 지표와 훈련으로 검증한다.
- 실패와 비용과 운영 책임이 없는 아키텍처 다이어그램은 완성된 설계가 아니다.

---

# 다음 Chapter

다음 Chapter에서는 [Scalability and High Availability](/aws-backend/part-16/09-scalability-high-availability/)를 학습한다.

앞 장의 결정을 다음 장의 더 구체적인 경계와 운영 절차로 연결한다.
