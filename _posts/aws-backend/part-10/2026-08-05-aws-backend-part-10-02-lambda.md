---
title: "Chapter 02. Lambda"
permalink: /aws-backend/part-10/02-lambda/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. Lambda
## 이벤트로 실행되는 함수

> **학습 목표**
>
> - Lambda 실행 모델을 설명할 수 있다.
> - Cold Start와 Timeout의 의미를 이해한다.
> - Lambda에 적합한 작업을 구분할 수 있다.

---

# Lambda란?

Lambda는 이벤트가 발생했을 때 함수를 실행하는 AWS 서비스다.

API Gateway, EventBridge, S3, SQS 같은 서비스가 Lambda를 호출할 수 있다.

---

# 실행 제약

- 실행 시간 제한
- 메모리 설정
- 임시 디스크 제한
- Cold Start
- 동시성 제한

---

# 적합한 작업

- 짧은 API 처리
- 이미지 리사이징
- 이벤트 후처리
- 스케줄 작업
- 작은 자동화 작업

---

# 기억해야 할 내용

Lambda는 짧고 독립적인 이벤트 처리에 적합하다.

긴 실행 작업이나 복잡한 상태ful 서버에는 맞지 않을 수 있다.


