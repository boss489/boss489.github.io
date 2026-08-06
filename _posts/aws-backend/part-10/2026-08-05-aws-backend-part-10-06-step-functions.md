---
title: "Chapter 06. Step Functions"
permalink: /aws-backend/part-10/06-step-functions/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 06. Step Functions
## 여러 작업을 상태 머신으로 연결하기

> **학습 목표**
>
> - Step Functions의 역할을 설명할 수 있다.
> - 여러 Lambda 작업을 순서대로 연결하는 방식을 이해한다.
> - 재시도, 분기, 보상 처리의 필요성을 설명할 수 있다.

---

# Step Functions란?

Step Functions는 여러 작업을 워크플로우로 연결하는 AWS 서비스다.

Lambda, ECS Task, Batch, SNS, SQS 등과 연결할 수 있다.

---

# 사용 예시

- 주문 처리 단계 관리
- 이미지 처리 파이프라인
- 승인 워크플로우
- 배치 작업 오케스트레이션

---

# 장점

- 흐름을 시각적으로 추적할 수 있다.
- 재시도와 실패 처리를 설정할 수 있다.
- 긴 작업을 여러 단계로 나눌 수 있다.

---

# 기억해야 할 내용

Step Functions는 복잡한 서버리스 흐름을 코드 안의 중첩 호출 대신 상태 머신으로 표현한다.


