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
## 파일을 객체로 저장하는 방식

> **학습 목표**
>
> - Object Storage의 개념을 설명할 수 있다.
> - S3가 파일 서버와 어떻게 다른지 이해한다.
> - 파일 업로드와 다운로드 흐름을 설명할 수 있다.

---

# Object Storage란?

Object Storage는 파일을 객체 단위로 저장하는 방식이다.

객체는 데이터와 메타데이터, 키를 가진다.

S3는 AWS의 대표 Object Storage 서비스다.

![S3 file service](/assets/aws-backend/s3-file-service.png)

---

# 파일 서버와 차이

일반 파일 서버는 디렉터리와 파일 시스템 중심이다.

S3는 Bucket 안에 Object Key로 파일을 저장한다.

애플리케이션 서버에 파일을 직접 저장하지 않으므로 서버 Scale Out에 유리하다.

---

# 기억해야 할 내용

S3는 서버 디스크가 아니라 독립적인 파일 저장소다.

백엔드 서버는 파일을 직접 들고 있기보다 S3에 저장하고 URL 또는 Key를 DB에 저장한다.


