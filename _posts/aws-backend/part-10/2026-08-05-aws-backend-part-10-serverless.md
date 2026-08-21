---
title: "Part 10. Serverless"
permalink: /aws-backend/part-10/
date: 2026-08-05T09:07:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 10. Serverless
## 서버리스 아키텍처

> 목표
>
> 서버를 직접 운영하지 않는 이벤트 기반 아키텍처를 이해한다.

---

# 학습 내용

- Lambda
- API Gateway
- EventBridge
- DynamoDB
- Step Functions

---

# 이 Part에서 연결할 이벤트 흐름

```text
HTTP Request → API Gateway → Lambda → DynamoDB
Domain Event  → EventBridge  → Lambda / Step Functions → 후속 작업
```

Serverless는 서버 운영이 사라지는 것이 아니라, 실행 단위와 실패 처리를 서비스 경계로 옮기는 방식이다. 동기 HTTP 요청, 비동기 이벤트, 여러 단계의 업무 흐름을 서로 다른 서비스로 나누어 선택한다.

# 실무 판단 기준

- Lambda는 짧고 독립적인 작업에 두고, 실행 시간·메모리·동시성 한계를 설계에 반영한다.
- 이벤트 소비자는 재전달을 전제로 멱등하게 만든다.
- DynamoDB는 테이블 구조보다 먼저 조회 패턴을 정한다.

---

# 완료 후 설명할 수 있어야 하는 것

- Lambda 실행 모델과 제약
- API Gateway와 Lambda를 연결하는 방식
- EventBridge로 이벤트를 라우팅하는 방식
- Step Functions로 작업 흐름을 구성하는 방식

---

다음 Part

→ **Part 11. Monitoring**
