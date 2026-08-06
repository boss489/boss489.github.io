---
title: "Chapter 02. DNS"
permalink: /aws-backend/part-04/02-dns/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. DNS
## 도메인 이름을 주소로 바꾸는 시스템

> **학습 목표**
>
> - DNS의 역할을 설명할 수 있다.
> - 도메인, 레코드, TTL의 의미를 이해한다.
> - 백엔드 장애에서 DNS를 확인해야 하는 이유를 설명할 수 있다.

---

# DNS란?

DNS(Domain Name System)는 사람이 읽는 도메인 이름을 네트워크가 사용할 수 있는 주소로 바꾸는 시스템이다.

예를 들어 `api.example.com`을 ALB 주소로 연결한다.

---

# 주요 레코드

| 레코드 | 역할 |
|---|---|
| A | 도메인을 IPv4 주소에 연결 |
| AAAA | 도메인을 IPv6 주소에 연결 |
| CNAME | 도메인을 다른 도메인에 연결 |
| Alias | Route 53에서 AWS 리소스에 연결 |

ALB는 IP가 바뀔 수 있으므로 Route 53 Alias를 사용하는 것이 일반적이다.

---

# TTL

TTL(Time To Live)은 DNS 응답을 캐시하는 시간이다.

TTL이 길면 DNS 변경 반영이 늦을 수 있고, TTL이 짧으면 DNS 조회가 더 자주 발생한다.

---

# 기억해야 할 내용

- DNS는 요청의 첫 출발점이다.
- ALB 연결에는 Route 53 Alias를 자주 사용한다.
- DNS 변경은 TTL 때문에 즉시 반영되지 않을 수 있다.


