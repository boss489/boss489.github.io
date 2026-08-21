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
> - DNS, Route 53, CloudFront, ALB, Target Group의 역할을 구분할 수 있다.
> - Health Check와 Auto Scaling이 요청 흐름에 미치는 영향을 이해한다.
> - 요청 실패 시 계층별로 원인을 추적할 수 있다.

---

# 왜 요청 흐름을 알아야 하는가

쇼핑몰의 상품 API가 응답하지 않는다고 가정해 보자.

Spring Boot 로그에 요청이 남지 않는다면 애플리케이션 코드만 확인해서는 원인을 찾을 수 없다.

도메인이 잘못된 주소를 가리키거나, ALB Listener가 요청을 받지 못하거나, Target이 비정상 상태일 수 있기 때문이다.

사용자가 입력한 `https://api.example.com/products`는 여러 네트워크 계층과 AWS 서비스를 통과한 뒤에야 애플리케이션에 도착한다.

따라서 백엔드 개발자는 **요청이 어디까지 도착했으며 어느 구간에서 끊겼는지** 설명할 수 있어야 한다.

---

# 전체 요청 흐름

일반적인 AWS 백엔드 요청 흐름은 다음과 같다.

```
Client
  │  1. api.example.com 조회
  ▼
DNS Resolver ───── Route 53
  │  2. 접속할 주소 응답
  ▼
CloudFront
  │  3. 캐시 확인 및 원본 요청
  ▼
ALB
  │  4. Listener와 Rule 평가
  ▼
Target Group
  │  5. 정상 Target 선택
  ▼
Spring Boot API
  │  6. 비즈니스 로직 실행
  ▼
Response
```

![Request flow](/assets/aws-backend/request-flow.png)

---

# 단계별로 살펴보기

## 1. Client가 도메인을 조회한다

브라우저나 모바일 앱은 먼저 `api.example.com`의 DNS 레코드를 조회한다.

로컬 캐시와 재귀 DNS Resolver에 유효한 응답이 없으면 권한 있는 DNS 서버까지 질의가 전달된다.

DNS의 해석 과정은 [DNS](/aws-backend/part-04/02-dns/)에서 자세히 다룬다.

## 2. Route 53이 접속 대상을 응답한다

Route 53 Hosted Zone은 도메인 레코드를 관리하고 ALB나 CloudFront 같은 AWS 리소스를 Alias로 연결한다.

라우팅 정책을 사용하면 가중치, 지연 시간, 장애 조치 같은 기준으로 응답 대상을 선택할 수 있다.

AWS에서의 DNS 운영은 [Route 53](/aws-backend/part-04/03-route53/)에서 살펴본다.

## 3. CloudFront가 캐시를 확인한다

CloudFront가 구성되어 있다면 가까운 Edge Location에서 요청을 먼저 처리한다.

캐시에 콘텐츠가 있으면 원본 서버까지 요청하지 않고 응답하며, 없으면 ALB 같은 Origin으로 전달한다.

동적 API도 TLS 종료, WAF 연동, 전 세계 전송 경로 최적화 목적으로 CloudFront를 통과할 수 있다.

## 4. ALB가 HTTP 요청을 분류한다

[ALB](/aws-backend/part-04/04-alb/)는 Listener에서 포트와 프로토콜을 확인하고 Rule을 우선순위대로 평가한다.

예를 들어 `/api/orders/*`는 주문 Target Group으로, `/api/products/*`는 상품 Target Group으로 전달할 수 있다.

## 5. Target Group이 정상 대상을 제공한다

[Target Group](/aws-backend/part-04/05-target-group/)은 EC2 인스턴스나 ECS Task처럼 실제 요청을 처리하는 대상을 묶는다.

ALB는 [Health Check](/aws-backend/part-04/06-health-check/) 결과가 정상인 Target으로 요청을 전달한다.

## 6. Spring Boot가 요청을 처리한다

선택된 서버의 애플리케이션 포트로 요청이 도착하면 Spring MVC의 `DispatcherServlet`이 컨트롤러를 찾아 비즈니스 로직을 실행한다.

처리 결과는 같은 경로를 반대로 거쳐 Client에 반환된다.

---

# 각 구성 요소의 책임

| 구성 요소 | 핵심 책임 | 대표 확인 항목 |
|---|---|---|
| DNS Resolver | 도메인 질의와 캐시 | 현재 응답, TTL |
| Route 53 | 권한 있는 DNS 응답 | Hosted Zone, Record, Policy |
| CloudFront | 엣지 캐시와 Origin 전달 | Cache Behavior, Origin |
| ALB | HTTP 수신과 L7 라우팅 | Listener, Rule, 인증서 |
| Target Group | 대상 등록과 상태 관리 | Target Type, Port, Health |
| Spring Boot | API와 비즈니스 로직 실행 | 애플리케이션 포트, 로그 |

각 계층은 서로 다른 문제를 해결하므로 한 서비스가 모든 책임을 대신하지 않는다.

---

# 네트워크 연결에서 확인할 것

요청 경로가 논리적으로 올바르더라도 네트워크 접근 규칙이 막혀 있으면 통신할 수 없다.

Internet-facing ALB는 Public Subnet에 배치하고, Spring Boot 서버는 일반적으로 Private Subnet에 배치한다.

```
VPC
├── Public Subnet
│   └── ALB Security Group
│       └── Inbound: HTTPS 443 from Internet
└── Private Subnet
    └── App Security Group
        └── Inbound: App Port from ALB Security Group
```

애플리케이션 Security Group은 인터넷 전체가 아니라 **ALB Security Group을 출발지로 허용**하는 것이 핵심이다.

Route Table, NACL, Security Group, 애플리케이션 수신 포트가 모두 연결되어야 요청이 완성된다.

---

# 응답이 돌아오는 과정

Spring Boot가 만든 HTTP 응답은 Target Group을 별도로 조회하지 않고 이미 만들어진 연결을 통해 ALB로 돌아간다.

ALB는 Client와의 연결에 응답을 전달하고, CloudFront가 있다면 캐시 정책에 따라 응답을 저장할 수 있다.

DNS는 연결 대상을 찾는 단계에 관여하며 개별 HTTP 응답을 중계하지 않는다.

| 구분 | DNS | ALB |
|---|---|---|
| 처리 대상 | 도메인 질의 | HTTP/HTTPS 요청 |
| 동작 시점 | 연결 전 주소 해석 | 연결 후 요청 처리 |
| 결과 | 접속할 이름 또는 주소 | 백엔드의 HTTP 응답 |
| 캐시 영향 | TTL | 연결 재사용, CloudFront 캐시와 별개 |

---

# Auto Scaling이 요청 흐름에 미치는 영향

[Auto Scaling](/aws-backend/part-04/07-auto-scaling/)은 트래픽에 따라 Target 수를 늘리거나 줄인다.

새 서버는 시작되었다는 이유만으로 즉시 요청을 받는 것이 아니라 Target Group에 등록되고 Health Check를 통과해야 한다.

Scale In 때는 연결 중인 요청을 마칠 시간을 주고 Target을 등록 해제해야 한다.

```
Metric 증가
  └── Scale Out
      └── 새 서버 시작
          └── Target 등록
              └── Health Check 통과
                  └── 요청 수신
```

세션을 서버 메모리에만 저장하면 서버 수가 변할 때 사용자 상태가 깨질 수 있으므로 [Session과 Keep Alive](/aws-backend/part-04/08-session-keep-alive/) 설계도 함께 고려해야 한다.

---

# 실무에서는 어떻게 사용할까

공개 쇼핑몰 API는 Route 53 Alias로 CloudFront나 ALB에 연결한다.

ALB는 HTTPS를 종료하고 `/api/*` 요청을 Spring Boot Target Group으로 전달한다.

Target Group은 여러 AZ의 서버를 포함하며, 비정상 서버를 요청 경로에서 제외한다.

트래픽이 증가하면 Auto Scaling이 서버를 추가하고 새 서버가 정상 상태가 된 뒤 분산 대상에 포함된다.

이 구조는 Part 3에서 배운 EC2와 Docker 애플리케이션을 단일 서버가 아닌 운영 가능한 서비스로 확장한다.

---

# 장애를 어디서부터 확인할까

장애 분석은 Client에 가까운 계층부터 애플리케이션 방향으로 진행하면 누락을 줄일 수 있다.

| 증상 | 우선 확인할 위치 | 대표 원인 |
|---|---|---|
| 도메인을 찾지 못함 | DNS, Route 53 | 레코드 누락, 위임 오류 |
| TLS 인증서 오류 | CloudFront, ALB Listener | 인증서 도메인 불일치 |
| 연결 시간 초과 | Route, Security Group | 포트 미허용, 경로 오류 |
| ALB 502 | Target, 애플리케이션 | 연결 종료, 잘못된 응답 |
| ALB 503 | Target Group | 정상 Target 없음 |
| ALB 504 | 애플리케이션 | 처리 지연, 타임아웃 |

Spring Boot Access Log에 요청이 없다면 DNS와 ALB 계층부터 확인한다.

Access Log에는 요청이 있지만 애플리케이션 오류가 발생한다면 Controller, 외부 API, 데이터베이스 같은 내부 의존성을 확인한다.

---

# 기억해야 할 내용

- 사용자의 요청은 DNS, Route 53, 선택적인 CloudFront, ALB, Target Group을 거쳐 Spring Boot에 도착한다.
- DNS는 주소를 해석하고 ALB는 HTTP 요청을 실제 서버로 전달한다.
- ALB Listener와 Rule은 요청의 진입점과 전달 조건을 결정한다.
- Target Group은 정상 Target만 요청 경로에 포함한다.
- Auto Scaling으로 추가된 서버는 Health Check를 통과한 뒤 요청을 받는다.
- Security Group과 애플리케이션 포트가 맞지 않으면 논리적 구성이 올바라도 통신할 수 없다.
- 장애는 Client에서 애플리케이션 방향으로 계층별 지표와 로그를 확인한다.

---

# 다음 Chapter

다음 Chapter에서는 요청의 첫 단계인 **DNS(Domain Name System)** 를 학습한다.

도메인이 주소로 해석되는 과정과 주요 레코드, TTL이 장애와 변경 작업에 미치는 영향을 자세히 살펴본다.

---

# 기억해야 할 내용

요청 흐름 장애는 한 지점만 보면 안 된다.

DNS, ALB Listener, Target Group, Security Group, 애플리케이션 포트를 순서대로 확인해야 한다.


