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
## 임시 업로드 권한 위임

> **학습 목표**
>
> - Presigned URL의 목적을 설명할 수 있다.
> - 서버를 거치지 않고 S3에 업로드하는 흐름을 이해한다.
> - 만료 시간과 권한 범위를 제한해야 하는 이유를 설명할 수 있다.

---

# Presigned URL이란?

Presigned URL은 제한된 시간 동안 S3 객체에 접근할 수 있는 서명된 URL이다.

백엔드는 업로드 권한을 직접 넘기지 않고 임시 URL만 발급한다.

![S3 file service](/assets/aws-backend/s3-file-service.png)

---

# 업로드 흐름

1. 브라우저가 백엔드에 업로드 URL을 요청한다.
2. 백엔드가 Presigned URL을 생성한다.
3. 브라우저가 S3로 직접 업로드한다.
4. 백엔드는 Object Key를 DB에 저장한다.

---

# 주의할 점

- 만료 시간을 짧게 설정한다.
- 업로드 가능한 파일 크기와 타입을 검증한다.
- Object Key를 예측하기 어렵게 만든다.
- 공개 권한을 열지 않는다.

---

# 기억해야 할 내용

Presigned URL은 임시 권한 위임 방식이다.

서버 부하를 줄이고 S3에 직접 업로드하게 만들 수 있다.


