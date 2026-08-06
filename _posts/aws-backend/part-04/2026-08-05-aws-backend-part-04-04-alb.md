---
title: "Chapter 04. ALB"
permalink: /aws-backend/part-04/04-alb/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. ALB
## HTTP 요청을 분산하는 로드밸런서

> **학습 목표**
>
> - ALB(Application Load Balancer)의 역할을 설명할 수 있다.
> - Load Balancer가 필요한 이유를 설명할 수 있다.
> - ALB가 Layer 7(Application Layer)에서 동작하는 이유를 이해한다.
> - Listener와 Rule의 관계를 이해한다.
> - Internet-facing ALB와 Internal ALB의 차이를 설명할 수 있다.
> - ALB가 Public Subnet에 배치되는 이유를 설명할 수 있다.

---

# ALB란?

ALB(Application Load Balancer)는 HTTP/HTTPS 요청을 여러 서버로 분산하는 AWS의 **L7(Application Layer) Load Balancer**이다.

사용자는 여러 서버의 존재를 알 필요 없이 하나의 ALB 주소만 바라본다. ALB는 들어온 요청을 적절한 서버로 전달하여 서비스의 안정성과 확장성을 높인다.

> 📷 **그림 4-1. ALB 개요**
>
> ```
> Browser
>    │
> Internet
>    │
>   ALB
>  ┌─┴───┐
> App  App
> ```

![ALB Overview](/assets/aws-backend/04-01-alb-overview.png)

---

# 왜 Load Balancer가 필요한가?

애플리케이션 서버가 한 대뿐이라면 모든 요청이 하나의 서버에 집중된다.

> 📷 **그림 4-2. 단일 서버 구조**
>
> ```
> Users
>   │
>  EC2
> ```

이 구조는 다음과 같은 문제가 있다.

- 서버 장애 시 서비스 전체가 중단된다.
- 트래픽이 증가하면 서버 한 대가 감당하기 어렵다.
- 서버를 추가해도 사용자가 어느 서버로 접속해야 하는지 알 수 없다.

로드밸런서는 이러한 문제를 해결하기 위해 하나의 진입점을 제공한다.

> 📷 **그림 4-3. Load Balancer 구조**
>
> ```
> Users
>   │
>  ALB
> ├── App1
> ├── App2
> └── App3
> ```

![Single server](/assets/aws-backend/04-02-single-server.png)

![Load balancer](/assets/aws-backend/04-03-load-balancer.png)

---

# ALB는 왜 L7 Load Balancer인가?

네트워크 장비는 처리하는 계층에 따라 구분된다.

| 구분 | 처리 대상 | 예 |
|------|-----------|----|
| L4 | TCP, UDP | NLB |
| L7 | HTTP, HTTPS | ALB |

ALB는 HTTP 요청을 이해할 수 있기 때문에 다음과 같은 정보를 기준으로 요청을 분기할 수 있다.

- URL Path
- Host
- HTTP Header
- Query String
- HTTP Method

> 📷 **그림 4-4. L4와 L7 비교**
>
> ```
> HTTP Request
> ┌────────────────────────┐
> GET /api/orders HTTP/1.1
> Host: api.example.com
> └────────────────────────┘
>
> L7(ALB)는 위 내용을 읽을 수 있다.
> L4는 TCP 연결만 전달한다.
> ```

![L4 and L7](/assets/aws-backend/04-04-l4-l7.png)

---

# Reverse Proxy

ALB는 Reverse Proxy 역할을 수행한다.

사용자는 실제 애플리케이션 서버에 직접 접속하지 않는다.

모든 요청은 먼저 ALB에 도착하고, ALB가 대신 서버로 요청을 전달한다.

> 📷 **그림 4-5. Reverse Proxy**
>
> ```
> Client
>    │
>   ALB
>    │
> Application Server
> ```

![Reverse proxy](/assets/aws-backend/04-05-reverse-proxy.png)

이를 통해 서버를 외부에 직접 노출하지 않아도 된다.

---

# Listener

Listener는 ALB가 **어떤 포트와 프로토콜로 요청을 받을지** 정의한다.

대표적인 Listener는 다음과 같다.

- HTTP : 80
- HTTPS : 443

HTTPS Listener를 사용하려면 ACM 인증서를 연결해야 한다.

> 📷 **그림 4-6. Listener**
>
> ```
> Internet
>    │
> HTTPS :443
>    │
> Listener
> ```

![Listener](/assets/aws-backend/04-06-listener.png)

Listener는 요청을 받은 뒤 Rule을 검사한다.

---

# Rule

Rule은 요청 조건에 따라 어느 대상으로 보낼지 결정하는 규칙이다.

대표적으로 다음 조건을 사용할 수 있다.

- Path
- Host
- Header
- Query String

예를 들어,

- `/api/*`
- `/admin/*`

처럼 URL 경로를 기준으로 서로 다른 애플리케이션으로 보낼 수 있다.

> 📷 **그림 4-7. Rule**
>
> ```
> Listener
>    │
> ┌──┴─────────┐
> /api/*  /admin/*
> ```

![Rule](/assets/aws-backend/04-07-rule.png)

> **참고**
>
> Rule이 실제로 요청을 전달하는 대상(Target Group)에 대해서는 다음 Chapter에서 자세히 살펴본다.

---

# Internet-facing ALB와 Internal ALB

ALB는 생성 시 두 가지 유형을 선택할 수 있다.

## Internet-facing

인터넷 사용자가 접근하는 ALB이다.

- 웹 서비스
- 모바일 API
- 공개 사이트

## Internal

VPC 내부에서만 접근 가능한 ALB이다.

- 내부 API
- MSA 간 통신
- 관리자 시스템

> 📷 **그림 4-8. Internet-facing vs Internal**
>
> ```
> Internet
>    │
> Public ALB
>
> VPC 내부
>    │
> Internal ALB
> ```

![Internet-facing vs Internal](/assets/aws-backend/04-08-facing-internal.png)

---

# ALB가 Public Subnet에 위치하는 이유

Internet-facing ALB는 인터넷으로부터 요청을 받아야 한다.

따라서 Internet Gateway와 연결된 Public Subnet에 배치한다.

반면 애플리케이션 서버는 일반적으로 외부에서 직접 접근할 필요가 없으므로 Private Subnet에 배치한다.

> 📷 **그림 4-9. 일반적인 배치**
>
> ```
> Internet
>    │
> IGW
>    │
> Public Subnet
>    │
> ALB
>    │
> Private Subnet
> App Servers
> ```

![ALB placement](/assets/aws-backend/04-09-alb-placement.png)

---

# ALB의 장점

- 하나의 진입점을 제공한다.
- HTTP 요청을 이해하고 다양한 조건으로 라우팅할 수 있다.
- 서버를 외부에 직접 노출하지 않아도 된다.
- HTTPS 종료(TLS Termination)를 수행할 수 있다.
- AWS의 다양한 서비스(ECS, EC2, Lambda)와 쉽게 연동된다.

> 📷 **그림 4-10. ALB 기능 요약**
>
> ```
> HTTPS
>   │
> Routing
>   │
> Reverse Proxy
>   │
> Backend
> ```

![ALB features](/assets/aws-backend/04-10-alb-features.png)

---

# 기억해야 할 내용

- ALB는 HTTP/HTTPS를 처리하는 Layer 7 Load Balancer이다.
- ALB는 하나의 진입점을 제공하여 여러 서버로 요청을 분산한다.
- Listener는 요청을 받는 입구이다.
- Rule은 요청을 어떤 조건으로 분기할지 정의한다.
- ALB는 Reverse Proxy 역할을 수행한다.
- Internet-facing ALB는 Public Subnet에 배치하고, Internal ALB는 VPC 내부에서만 사용한다.

---

# 다음 Chapter

다음 장에서는 **Target Group**을 학습한다.

Target Group이 무엇이며 ALB가 실제 서버를 어떻게 선택하는지 자세히 살펴본다.

