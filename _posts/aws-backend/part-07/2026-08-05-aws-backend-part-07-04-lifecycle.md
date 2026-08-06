---
title: "Chapter 04. Lifecycle"
permalink: /aws-backend/part-07/04-lifecycle/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Lifecycle
## 오래된 객체를 자동으로 정리하기

> **학습 목표**
>
> - Lifecycle 정책의 목적을 설명할 수 있다.
> - Storage Class 전환과 만료의 차이를 이해한다.
> - 파일 보관 기간을 비용과 함께 설계할 수 있다.

---

# Lifecycle이란?

Lifecycle은 S3 객체를 일정 조건에 따라 자동으로 이동하거나 삭제하는 정책이다.

---

# 대표 동작

- 오래된 객체 삭제
- 오래된 버전 삭제
- Storage Class 변경
- Multipart Upload 미완료 조각 정리

---

# 사용 예시

로그 파일은 30일 뒤 저렴한 스토리지로 이동하고 1년 뒤 삭제할 수 있다.

임시 업로드 파일은 하루 뒤 삭제할 수 있다.

---

# 기억해야 할 내용

Lifecycle은 비용 관리 도구다.

파일 성격별 보관 기간을 정하고 자동 정리해야 한다.


