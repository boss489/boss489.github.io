---
title: "Chapter 06. Multipart Upload"
permalink: /aws-backend/part-07/06-multipart-upload/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Multipart Upload
## 큰 파일을 나누어 업로드하기

> **학습 목표**
>
> - Multipart Upload의 목적을 설명할 수 있다.
> - 큰 파일 업로드 실패 시 재시도 방식을 이해한다.
> - 미완료 업로드 정리가 필요한 이유를 설명할 수 있다.

---

# Multipart Upload란?

Multipart Upload는 큰 파일을 여러 조각으로 나누어 업로드하는 방식이다.

일부 조각만 실패하면 실패한 조각만 다시 업로드할 수 있다.

---

# 장점

- 큰 파일 업로드 안정성 증가
- 병렬 업로드 가능
- 실패 시 재시도 범위 축소

---

# 주의할 점

업로드를 시작하고 완료하지 않은 조각도 비용이 발생할 수 있다.

Lifecycle 정책으로 미완료 Multipart Upload를 정리해야 한다.

---

# 기억해야 할 내용

큰 파일은 Multipart Upload가 유리하다.

미완료 업로드 정리 정책을 반드시 함께 둔다.


