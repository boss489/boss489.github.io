---
title: "Chapter 05. Ingress"
permalink: /aws-backend/part-09/05-ingress/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Ingress
## HTTP 요청을 Service로 라우팅하기

> **학습 목표**
>
> - Ingress의 역할을 설명할 수 있다.
> - Host와 Path 기반 라우팅을 이해한다.
> - Ingress Controller가 필요한 이유를 설명할 수 있다.

---

# Ingress란?

Ingress는 외부 HTTP/HTTPS 요청을 클러스터 내부 Service로 라우팅하는 규칙이다.

예시는 다음과 같다.

- `api.example.com` -> API Service
- `/admin` -> Admin Service

---

# Ingress Controller

Ingress는 규칙이고, 실제 로드밸런서를 만들고 트래픽을 처리하는 것은 Ingress Controller다.

AWS에서는 ALB Controller를 많이 사용한다.

---

# 기억해야 할 내용

Ingress는 HTTP 라우팅 규칙이다.

실제로 동작하려면 Controller가 필요하다.


