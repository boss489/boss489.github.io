---
title: "Chapter 02. Bucket and Object"
permalink: /aws-backend/part-07/02-bucket-object/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Bucket and Object
## S3의 기본 단위

> **학습 목표**
>
> - Bucket과 Object의 관계를 설명할 수 있다.
> - Object Key의 의미를 이해한다.
> - S3 권한 설정의 기본 단위를 이해한다.

---

# Bucket

Bucket은 S3 객체를 담는 최상위 컨테이너다.

Bucket 이름은 전역에서 고유해야 한다.

---

# Object

Object는 S3에 저장되는 파일 단위다.

Object는 다음을 가진다.

- Key
- Data
- Metadata
- Version ID
- Storage Class

---

# Object Key

Object Key는 객체의 이름이다.

`images/2026/08/sample.png`처럼 경로처럼 보이지만 실제 디렉터리는 아니다.

---

# 기억해야 할 내용

Bucket은 저장 공간이고 Object는 저장되는 파일이다.

권한, Lifecycle, Versioning은 Bucket 단위로 많이 설정한다.


