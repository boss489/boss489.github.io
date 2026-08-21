---
title: "Part 7. Object Storage"
permalink: /aws-backend/part-07/
date: 2026-08-05T09:10:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 7. Object Storage
## S3 기반 파일 서비스

> 목표
>
> S3와 CloudFront를 이용한 파일 저장 및 제공 구조를 이해한다.

---

# 학습 내용

- Bucket
- Object
- Versioning
- Lifecycle
- Presigned URL
- Multipart Upload
- CloudFront

---

# 이 Part에서 다룰 파일 흐름

```text
Browser → Spring Boot에서 Presigned URL 발급 → S3 직접 업로드
Browser → CloudFront → S3 Origin → 파일 응답
```

애플리케이션 서버가 대용량 파일의 중계자가 되면 서버 메모리, 연결, 확장 비용을 모두 감당해야 한다. 업로드 권한 발급과 파일 전송을 분리하는 이유가 여기에 있다.

# 실무 판단 기준

- Bucket을 공개로 열기보다 CloudFront와 최소 권한 정책으로 제공 범위를 제한한다.
- Presigned URL은 대상 Key, HTTP 메서드, 만료 시간을 좁혀 발급한다.
- Versioning을 켰다면 Lifecycle로 오래된 버전과 미완료 Multipart Upload 정리도 함께 설계한다.

---

# 완료 후 설명할 수 있어야 하는 것

- Bucket과 Object의 관계
- Presigned URL로 업로드 권한을 위임하는 방식
- Lifecycle 정책으로 비용을 줄이는 방식
- CloudFront로 S3 파일을 제공하는 구조

---

다음 Part

→ **Part 8. Container Platform**
