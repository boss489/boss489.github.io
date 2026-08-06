---
title: "Chapter 07. Auto Scaling"
permalink: /aws-backend/part-04/07-auto-scaling/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. Auto Scaling
## 트래픽 변화에 맞춰 서버 수 조절하기

> **학습 목표**
>
> - Auto Scaling의 목적을 설명할 수 있다.
> - Desired, Min, Max Capacity의 의미를 이해한다.
> - Scaling Policy가 어떤 지표를 기준으로 동작하는지 설명할 수 있다.

---

# Auto Scaling이란?

Auto Scaling은 부하에 따라 서버나 Task 수를 자동으로 조절하는 기능이다.

트래픽이 늘면 인스턴스나 Task를 늘리고, 줄면 다시 줄인다.

---

# Capacity

| 값 | 의미 |
|---|---|
| Min | 최소 유지 수 |
| Desired | 현재 유지하려는 수 |
| Max | 최대 확장 수 |

Max를 너무 낮게 잡으면 트래픽 증가에 대응하지 못한다.

---

# Scaling Policy

Scaling Policy는 어떤 조건에서 늘리고 줄일지 정한다.

예시는 다음과 같다.

- CPU 사용률
- 메모리 사용률
- ALB Request Count
- Queue Length

---

# 기억해야 할 내용

- Auto Scaling은 가용성과 비용을 함께 다룬다.
- Min, Desired, Max를 구분해야 한다.
- 지표 선택이 잘못되면 필요한 순간에 확장되지 않는다.


