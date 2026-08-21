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
> - 재귀 질의와 권한 있는 DNS 응답 과정을 이해한다.
> - 주요 DNS 레코드와 TTL의 의미를 설명할 수 있다.
> - 백엔드 장애에서 DNS를 확인해야 하는 이유를 설명할 수 있다.

---

# 왜 DNS가 필요한가

쇼핑몰 API 서버의 IP 주소가 `203.0.113.10`이라고 가정해 보자.

사용자가 숫자 주소를 기억하는 것은 어렵고 서버 교체로 주소가 바뀔 때마다 앱 설정을 수정할 수도 없다.

여러 서버를 제공하는 ALB는 고정된 단일 IP 주소가 아니라 DNS 이름으로 노출되므로 IP를 직접 등록하는 방식도 적합하지 않다.

DNS는 `api.example.com`이라는 안정적인 이름과 현재 접속해야 할 대상을 분리한다.

백엔드 인프라가 바뀌어도 DNS 레코드만 변경하면 사용자는 같은 도메인을 계속 사용할 수 있다.

---

# DNS란?

DNS(Domain Name System)는 사람이 읽는 도메인 이름을 네트워크가 사용할 수 있는 주소로 바꾸는 시스템이다.

DNS는 전 세계에 분산된 계층형 데이터베이스이며 하나의 중앙 서버가 모든 도메인을 관리하지 않는다.

예를 들어 `api.example.com`을 ALB 주소로 연결하면 Client는 도메인을 조회한 뒤 응답받은 대상으로 HTTPS 연결을 시작한다.

```
api.example.com
        │ DNS 조회
        ▼
ALB DNS Name 또는 IP 주소
        │ HTTPS 연결
        ▼
Spring Boot API
```

DNS는 HTTP 요청 내용을 전달하지 않고 **연결할 대상을 찾는 역할**만 한다.

---

# 도메인의 계층 구조

DNS 이름은 오른쪽에서 왼쪽으로 범위가 구체화된다.

```
.
└── com
    └── example
        ├── www
        └── api
```

`api.example.com`에서 `com`은 최상위 도메인, `example.com`은 등록 도메인, `api`는 하위 도메인이다.

마지막의 루트 `.`은 일반적으로 생략한다.

이 계층 덕분에 각 조직은 자신에게 위임된 도메인의 하위 레코드를 독립적으로 관리할 수 있다.

---

# DNS 조회는 어떻게 동작하는가

브라우저가 `api.example.com`을 조회할 때 일반적인 흐름은 다음과 같다.

```
Client
  │ 1. 질의
  ▼
Recursive Resolver
  ├── 2. Root Name Server
  ├── 3. .com Name Server
  └── 4. example.com Authoritative Server
          │
          └── 5. 최종 레코드 응답
```

Client는 보통 통신사나 회사, 공개 DNS의 Recursive Resolver에 질의를 맡긴다.

Resolver는 캐시에 답이 있으면 즉시 반환하고, 없으면 Root부터 권한 있는 Name Server까지 순서대로 찾아간다.

권한 있는 DNS 서버는 자신이 관리하는 Zone의 최종 레코드를 응답한다.

AWS에서는 [Route 53](/aws-backend/part-04/03-route53/) Hosted Zone이 권한 있는 DNS 역할을 할 수 있다.

---

# 주요 DNS 레코드

| 레코드 | 역할 | 예 |
|---|---|---|
| A | 이름을 IPv4 주소에 연결 | `api.example.com → 203.0.113.10` |
| AAAA | 이름을 IPv6 주소에 연결 | `api.example.com → 2001:db8::10` |
| CNAME | 이름을 다른 정규 이름에 연결 | `www → site.example.net` |
| MX | 메일을 받을 서버 지정 | 메일 서비스 연결 |
| TXT | 문자열 기반 검증과 정책 저장 | 도메인 소유권 검증 |
| NS | Zone을 담당하는 Name Server 지정 | 도메인 위임 |
| SOA | Zone의 기본 관리 정보 제공 | Serial, 기본 서버 정보 |

Alias는 표준 DNS 레코드 타입이 아니라 Route 53이 제공하는 확장 기능이다.

ALB는 내부 IP가 바뀔 수 있으므로 Route 53에서는 IP를 직접 적지 않고 Alias로 연결하는 것이 일반적이다.

---

# A, CNAME, Alias 비교

| 항목 | A | CNAME | Route 53 Alias |
|---|---|---|---|
| 값 | IPv4 주소 | 다른 도메인 이름 | 지원되는 AWS 리소스 |
| Zone Apex 사용 | 가능 | 일반적으로 불가 | 가능 |
| ALB 연결 적합성 | 낮음 | 하위 도메인에서 가능 | 높음 |
| AWS 전용 기능 | 아님 | 아님 | 맞음 |

Zone Apex는 `example.com`처럼 하위 이름이 없는 루트 도메인을 의미한다.

Alias를 사용하면 `example.com`도 ALB나 CloudFront 같은 지원 대상에 연결할 수 있다.

---

# TTL

TTL(Time To Live)은 DNS 응답을 캐시하는 시간이다.

TTL이 길면 DNS 변경 반영이 늦을 수 있고, TTL이 짧으면 DNS 조회가 더 자주 발생한다.

```
권한 있는 DNS: 새 주소로 변경
        │
        ├── TTL 만료된 Resolver → 새 주소
        └── TTL 남은 Resolver   → 이전 주소
```

TTL은 변경 후 기다리는 시간이 아니라 **응답을 받은 시점부터 캐시할 수 있는 시간**이다.

DNS 이전이나 장애 조치를 계획한다면 변경 전에 TTL을 낮추고 기존 캐시가 만료될 시간을 확보해야 한다.

변경 직전에 TTL을 낮춰도 이미 긴 TTL로 저장된 캐시는 즉시 사라지지 않는다.

---

# CLI에서는

`dig`를 사용하면 현재 DNS 응답과 TTL을 확인할 수 있다.

```bash
dig api.example.com
dig api.example.com A +short
dig api.example.com CNAME
```

권한 있는 Name Server를 확인하려면 NS 레코드를 조회한다.

```bash
dig example.com NS +short
```

`+trace`는 Root부터 위임 경로를 따라가므로 Name Server 위임 문제를 분석할 때 유용하다.

```bash
dig api.example.com +trace
```

---

# 실무에서는 어떻게 사용할까

운영 API는 `api.example.com`, 관리자 도구는 `admin.example.com`처럼 역할별로 하위 도메인을 분리한다.

도메인은 인프라 주소와 애플리케이션 설정을 분리하는 안정적인 계약으로 사용한다.

외부 API URL을 Spring Boot 설정에 저장할 때도 변경 가능한 도메인을 사용하고 특정 IP에 의존하지 않는다.

배포 전환 시에는 DNS만으로 즉시 트래픽을 바꾸려 하기보다 ALB의 Rule이나 Target Group 전환과 함께 선택한다.

DNS 캐시는 Client마다 만료 시점이 다르므로 짧은 시간 동안 이전 대상과 새 대상이 함께 요청을 받을 수 있다.

---

# 장애 사례

## 레코드는 만들었지만 조회되지 않는 경우

도메인 등록 기관의 NS 설정이 실제 Hosted Zone의 Name Server와 다르면 Route 53 레코드는 사용되지 않는다.

`dig NS` 결과와 Hosted Zone에 할당된 Name Server를 비교해야 한다.

## 변경했지만 이전 서버로 요청되는 경우

Resolver나 운영체제, 브라우저에 이전 응답이 TTL 동안 남아 있을 수 있다.

여러 Resolver에서 조회하고 TTL이 만료될 때까지 이전 서버도 정상적으로 유지해야 한다.

## `NXDOMAIN`이 발생하는 경우

레코드 이름 오타, 잘못된 Zone, CNAME 연결 대상 누락을 확인한다.

실패 응답도 일정 시간 캐시될 수 있으므로 레코드를 복구한 뒤 즉시 모든 Client에서 정상화되지 않을 수 있다.

## DNS는 정상이지만 API가 실패하는 경우

`dig`가 올바른 대상을 반환한다면 [ALB](/aws-backend/part-04/04-alb/) Listener, Security Group, Target 상태를 이어서 확인한다.

DNS 응답 성공은 애플리케이션 정상 동작을 보장하지 않는다.

---

# 기억해야 할 내용

- DNS는 도메인 이름을 접속 가능한 대상으로 해석하는 분산 시스템이다.
- Resolver는 캐시에 답이 없으면 Root부터 권한 있는 DNS 서버까지 조회한다.
- A와 AAAA는 IP 주소를, CNAME은 다른 도메인 이름을 가리킨다.
- Route 53 Alias는 Zone Apex와 AWS 리소스 연결에 유용하다.
- TTL 때문에 DNS 변경 전후에 이전 응답과 새 응답이 함께 사용될 수 있다.
- DNS 조회 성공과 HTTP 요청 성공은 별개의 문제이다.
- 장애 시 `dig`로 응답 값, TTL, NS 위임을 순서대로 확인한다.

---

# 다음 Chapter

다음 Chapter에서는 AWS의 권한 있는 DNS 서비스인 **Route 53**을 학습한다.

Hosted Zone과 Alias Record를 구성하는 방법, 트래픽 목적에 맞는 라우팅 정책을 자세히 살펴본다.

