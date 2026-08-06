---
title: "Chapter 01. Request Flow Overview"
permalink: /aws-backend/part-04/01-request-flow-overview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. Request Flow Overview
## 인터넷 요청이 API까지 가는 길

> **학습 목표**
>
> - 사용자의 요청이 백엔드까지 도달하는 전체 흐름을 설명할 수 있다.
> - DNS, Route 53, ALB, Target Group의 역할을 구분할 수 있다.
> - Health Check와 Auto Scaling이 요청 흐름에 미치는 영향을 이해한다.

---

# 전체 흐름

사용자가 브라우저에서 도메인을 입력하면 요청은 바로 Spring Boot로 가지 않는다.

일반적인 AWS 백엔드 요청 흐름은 다음과 같다.

![Request flow](/assets/aws-backend/request-flow.png)

```
User
  -> DNS
  -> Route 53
  -> ALB
  -> Target Group
  -> Spring Boot
```

---

# 핵심 포인트

- DNS는 도메인을 접속 가능한 대상으로 바꾼다.
- Route 53은 AWS의 DNS 서비스다.
- ALB는 HTTP/HTTPS 요청을 여러 서버로 분산한다.
- Target Group은 ALB가 요청을 보낼 대상 묶음이다.
- Health Check가 실패한 대상에는 요청을 보내지 않는다.

---

# 기억해야 할 내용

요청 흐름 장애는 한 지점만 보면 안 된다.

DNS, ALB Listener, Target Group, Security Group, 애플리케이션 포트를 순서대로 확인해야 한다.


