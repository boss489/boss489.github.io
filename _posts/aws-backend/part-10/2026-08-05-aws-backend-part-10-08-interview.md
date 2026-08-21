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

## 동기 API와 EventBridge 이벤트는 어떻게 선택하나요?

호출자가 즉시 결과를 받아야 하면 동기 API를 사용합니다. 후속 작업을 느슨하게 연결하거나 실패 재처리를 분리해야 하면 이벤트를 발행하고 소비자가 처리하게 합니다.

## Lambda의 Cold Start를 어떻게 다루나요?

실행 패키지와 초기화 작업을 작게 유지하고, 실제 지연 시간이 문제인지 측정합니다. 항상 낮은 지연이 필요한 경로라면 Provisioned Concurrency나 다른 실행 모델을 검토합니다.

## 이벤트 소비자가 멱등해야 하는 이유는 무엇인가요?

이벤트는 재시도나 중복 전달될 수 있습니다. 같은 이벤트를 다시 처리해도 결과가 한 번 처리한 것과 같도록 이벤트 ID 기록, 조건부 쓰기, 중복 방지 Key를 사용합니다.

