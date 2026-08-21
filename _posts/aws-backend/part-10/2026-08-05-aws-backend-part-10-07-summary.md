---
title: "Chapter 07. Part 10 Summary"
permalink: /aws-backend/part-10/07-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Part 10 Summary
## Serverless 핵심 정리

![Serverless flow](/assets/aws-backend/serverless-flow.png)

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| Lambda | 이벤트 기반 함수 실행 |
| API Gateway | HTTP API 입구 |
| EventBridge | 이벤트 라우팅 |
| DynamoDB | 서버리스 NoSQL DB |
| Step Functions | 상태 머신 기반 워크플로우 |

---

# 실무 포인트

- Lambda는 짧고 독립적인 작업에 적합하다.
- API Gateway Timeout과 Lambda Timeout을 함께 고려한다.
- 이벤트 기반 구조는 실패 처리와 재처리가 중요하다.
- DynamoDB는 조회 패턴 중심으로 설계한다.

---

# 요청 성격별 선택

| 상황 | 기본 선택 | 이유 |
|---|---|---|
| 짧은 HTTP API | API Gateway + Lambda | 요청 단위로 실행·확장한다 |
| 서비스 간 비동기 알림 | EventBridge | 발행자와 소비자를 느슨하게 연결한다 |
| 순서·분기·재시도가 있는 업무 | Step Functions | 흐름과 실패 상태를 눈에 보이게 관리한다 |
| 예측 가능한 Key 조회 | DynamoDB | 서버 운영 없이 낮은 지연으로 조회한다 |

서버리스 설계에서는 성공 경로만큼 재시도, 중복 전달, 타임아웃, 부분 실패 뒤의 동작을 먼저 정해야 한다.

