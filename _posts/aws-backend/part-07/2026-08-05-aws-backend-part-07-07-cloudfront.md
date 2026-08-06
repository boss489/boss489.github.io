---
title: "Chapter 07. CloudFront"
permalink: /aws-backend/part-07/07-cloudfront/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. CloudFront
## S3 파일을 빠르게 제공하기

> **학습 목표**
>
> - CloudFront의 역할을 설명할 수 있다.
> - S3와 CloudFront를 함께 쓰는 이유를 이해한다.
> - 캐시와 원본 접근 제어의 기본 개념을 설명할 수 있다.

---

# CloudFront란?

CloudFront는 AWS의 CDN 서비스다.

사용자 가까운 Edge Location에서 파일을 제공해 지연 시간을 줄인다.

---

# S3와 함께 쓰는 이유

S3를 직접 공개하지 않고 CloudFront를 통해 파일을 제공할 수 있다.

캐시를 활용하면 S3 요청 수와 응답 시간을 줄일 수 있다.

---

# 운영 포인트

- Cache Policy
- Origin Access Control
- HTTPS 인증서
- Invalidations
- Signed URL 또는 Signed Cookie

---

# 기억해야 할 내용

CloudFront는 S3 앞단 CDN이다.

공개 파일 제공, 다운로드 성능, 원본 보호를 함께 고려한다.


