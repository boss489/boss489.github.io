---
title: "Chapter 08. Part 7 Summary"
permalink: /aws-backend/part-07/08-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Part 7 Summary
## Object Storage 핵심 정리

![S3 file service](/assets/aws-backend/s3-file-service.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| Bucket | 객체를 담는 저장 공간 |
| Object | S3에 저장되는 파일 단위 |
| Versioning | 객체 변경 이력 보관 |
| Lifecycle | 오래된 객체 자동 정리 |
| Presigned URL | 임시 접근 권한 위임 |
| Multipart Upload | 큰 파일 분할 업로드 |
| CloudFront | S3 파일 CDN 제공 |

---

# 실무 포인트

- 서버 디스크에 파일을 저장하지 않는다.
- 업로드는 Presigned URL로 S3에 직접 보내는 구조가 단순하다.
- 공개 파일은 CloudFront로 제공한다.
- Lifecycle로 비용을 관리한다.

---

# 파일 서비스 기본 경계

```text
인증된 사용자 → API의 업로드 권한 확인 → Presigned URL
브라우저       → S3 직접 업로드
일반 사용자   → CloudFront로 배포된 파일 조회
```

업로드 권한 판단은 API가 담당하고, 파일 바이트 전송은 S3가 담당한다. 이 경계를 지키면 서버 확장과 대용량 업로드를 단순하게 처리할 수 있다.

# 운영 확인 항목

- URL 만료 시간과 업로드 가능한 Key Prefix를 제한했는가?
- 삭제된 객체와 이전 버전의 보관 기간을 정했는가?
- CloudFront 캐시 무효화보다 파일명 버전 전략으로 갱신할 수 있는가?

