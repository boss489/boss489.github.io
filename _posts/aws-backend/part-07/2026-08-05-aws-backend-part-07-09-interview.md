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


