---
title: "Chapter 08. Interview Questions"
permalink: /aws-backend/part-10/08-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. Interview Questions
## Serverless 면접 질문

---

## Serverless란 무엇인가요?

서버가 없다는 뜻이 아니라 사용자가 서버를 직접 프로비저닝하고 운영하지 않는 실행 모델입니다.

## Lambda의 장점과 제약은 무엇인가요?

장점은 이벤트 기반 실행과 사용량 기반 과금입니다. 제약은 실행 시간 제한, Cold Start, 동시성 제한입니다.

## API Gateway는 어떤 역할을 하나요?

HTTP 요청을 받아 Lambda 같은 백엔드로 전달하는 서버리스 API의 입구 역할을 합니다.

## EventBridge는 언제 사용하나요?

서비스 간 이벤트를 느슨하게 연결하고, 규칙에 따라 여러 대상으로 라우팅할 때 사용합니다.

## DynamoDB 설계에서 중요한 것은 무엇인가요?

조회 패턴을 먼저 정하고 Partition Key와 Sort Key를 설계하는 것입니다.

## Step Functions는 왜 사용하나요?

여러 작업의 순서, 분기, 재시도, 실패 처리를 상태 머신으로 관리하기 위해 사용합니다.


