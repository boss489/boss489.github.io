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


