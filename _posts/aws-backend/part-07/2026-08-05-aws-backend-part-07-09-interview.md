---
title: "Chapter 09. Interview Questions"
permalink: /aws-backend/part-07/09-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Interview Questions
## Object Storage 면접 질문

---

## S3의 Bucket과 Object는 무엇인가요?

Bucket은 객체를 담는 저장 공간이고, Object는 실제 저장되는 파일 단위입니다.

## Presigned URL은 왜 사용하나요?

임시 접근 권한을 발급해 브라우저가 S3에 직접 업로드하거나 다운로드할 수 있게 하기 위해 사용합니다.

## Versioning의 장점과 단점은 무엇인가요?

삭제나 덮어쓰기 복구가 가능하지만 오래된 버전이 쌓여 비용이 증가할 수 있습니다.

## Lifecycle은 왜 필요한가요?

오래된 객체나 미완료 업로드를 자동으로 정리해 비용을 줄이기 위해 필요합니다.

## CloudFront를 S3 앞에 두는 이유는 무엇인가요?

사용자 가까운 Edge에서 파일을 제공해 속도를 높이고, S3 원본을 직접 공개하지 않기 위해 사용합니다.

## S3 Bucket을 Public으로 열지 않는 이유는 무엇인가요?

의도하지 않은 파일까지 인터넷에 노출될 수 있기 때문입니다. 공개 제공이 필요하면 CloudFront를 경유하고, 업로드·관리 권한은 IAM 정책과 Presigned URL로 최소화합니다.

## 파일을 교체했는데 CloudFront에서 이전 파일이 보이면 어떻게 하나요?

캐시 TTL이 남아 있을 수 있습니다. 같은 URL을 반드시 유지해야 한다면 Invalidation을 사용하고, 가능하면 파일명이나 경로에 버전을 넣어 새 URL로 배포합니다.

## Multipart Upload는 언제 필요하나요?

큰 파일을 여러 조각으로 나누어 전송하고, 중단된 업로드를 재개해야 할 때 사용합니다. 완료되지 않은 Upload는 Lifecycle 규칙으로 정리해 비용이 남지 않게 합니다.

