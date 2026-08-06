---
title: "Chapter 03. Route 53"
permalink: /aws-backend/part-04/03-route53/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Route 53
## AWS의 DNS 서비스

> **학습 목표**
>
> - Route 53 Hosted Zone의 의미를 설명할 수 있다.
> - Alias Record가 AWS 리소스 연결에 왜 유용한지 이해한다.
> - DNS 라우팅 정책의 기본 목적을 설명할 수 있다.

---

# Hosted Zone

Hosted Zone은 도메인의 DNS 레코드를 관리하는 공간이다.

예를 들어 `example.com` Hosted Zone 안에 `api.example.com` 레코드를 만든다.

---

# Alias Record

Alias Record는 Route 53에서 AWS 리소스를 직접 가리키는 레코드다.

ALB, CloudFront, S3 Website Endpoint 같은 리소스에 연결할 때 사용한다.

CNAME과 달리 루트 도메인에도 사용할 수 있다.

---

# 라우팅 정책

Route 53은 단순 연결 외에도 여러 라우팅 정책을 제공한다.

- Simple
- Weighted
- Latency
- Failover
- Geolocation

처음에는 Simple과 Alias만 이해해도 충분하다.

---

# 기억해야 할 내용

- Route 53은 AWS DNS 서비스다.
- Hosted Zone은 도메인의 DNS 설정 공간이다.
- ALB 연결에는 Alias Record를 사용한다.


