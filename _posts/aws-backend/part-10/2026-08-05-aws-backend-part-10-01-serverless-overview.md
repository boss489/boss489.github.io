---
title: "Chapter 01. Serverless Overview"
permalink: /aws-backend/part-10/01-serverless-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Serverless Overview
## 서버를 직접 관리하지 않는 실행 모델

> **학습 목표**
>
> - Serverless의 의미를 설명할 수 있다.
> - Lambda, API Gateway, EventBridge의 관계를 이해한다.
> - 서버리스가 적합한 경우와 그렇지 않은 경우를 구분할 수 있다.

---

# Serverless란?

Serverless는 서버가 없다는 뜻이 아니라 서버 운영을 사용자가 직접 하지 않는 모델이다.

AWS가 실행 환경을 관리하고 사용자는 함수, 이벤트, 권한, 데이터 흐름을 설계한다.

![Serverless flow](/assets/aws-backend/serverless-flow.png)

---

# 장점

- 서버 프로비저닝이 필요 없다.
- 사용량 기반 과금이다.
- 이벤트 기반 처리에 적합하다.
- 작은 기능을 빠르게 만들 수 있다.

---

# 한계

- 실행 시간 제한
- Cold Start
- 로컬 디버깅 어려움
- 복잡한 흐름 추적 어려움
- 상태 관리 제약

---

# 기억해야 할 내용

Serverless는 운영 부담을 줄이지만 설계 복잡도를 없애지는 않는다.


