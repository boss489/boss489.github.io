---
title: "Chapter 04. EventBridge"
permalink: /aws-backend/part-10/04-eventbridge/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. EventBridge
## 이벤트를 라우팅하는 버스

> **학습 목표**
>
> - EventBridge의 역할을 설명할 수 있다.
> - Event Bus와 Rule의 관계를 이해한다.
> - 이벤트 기반 아키텍처의 장단점을 설명할 수 있다.

---

# EventBridge란?

EventBridge는 이벤트를 받아 규칙에 따라 여러 대상으로 전달하는 서비스다.

예를 들어 주문 완료 이벤트를 받아 쿠폰, 알림, 정산 시스템으로 전달할 수 있다.

---

# 구성 요소

- Event Bus
- Event
- Rule
- Target
- Schema

---

# 장점

서비스 간 직접 의존을 줄일 수 있다.

새로운 후처리를 추가할 때 기존 서비스 코드를 덜 건드릴 수 있다.

---

# 기억해야 할 내용

EventBridge는 이벤트 라우터다.

중요 이벤트는 재처리와 실패 처리까지 함께 설계해야 한다.


