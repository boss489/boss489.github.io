---
title: "Chapter 03. API Gateway"
permalink: /aws-backend/part-10/03-api-gateway/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. API Gateway
## HTTP 요청을 Lambda로 연결하기

> **학습 목표**
>
> - API Gateway의 역할을 설명할 수 있다.
> - Lambda Proxy Integration 흐름을 이해한다.
> - 인증, 제한, 로깅 기능을 설명할 수 있다.

---

# API Gateway란?

API Gateway는 HTTP API를 만들고 백엔드 Lambda나 서비스로 요청을 전달하는 AWS 서비스다.

서버리스 API의 앞단 역할을 한다.

![Serverless flow](/assets/aws-backend/serverless-flow.png)

---

# 주요 기능

- 라우팅
- 인증 연동
- Rate Limit
- Request/Response 변환
- Access Log
- Custom Domain

---

# Lambda Proxy

Lambda Proxy Integration은 HTTP 요청 정보를 Lambda 이벤트로 전달하고, Lambda 응답을 HTTP 응답으로 돌려준다.

---

# 기억해야 할 내용

API Gateway는 서버리스 API의 입구다.

Lambda와 함께 쓸 때 요청/응답 포맷과 Timeout을 주의해야 한다.


