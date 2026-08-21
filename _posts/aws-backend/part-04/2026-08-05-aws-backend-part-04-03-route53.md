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
> - Public Hosted Zone과 Private Hosted Zone을 구분할 수 있다.
> - DNS 라우팅 정책을 요구사항에 맞게 선택할 수 있다.

---

# 왜 Route 53이 필요한가

쇼핑몰의 Spring Boot API를 ALB 뒤에 배포했다고 가정해 보자.

ALB가 제공하는 긴 DNS 이름을 사용자에게 직접 노출하면 기억하기 어렵고 인프라 교체에도 취약하다.

`api.example.com`을 ALB에 연결하고 장애 시 다른 리전으로 전환하려면 DNS 레코드를 안정적으로 관리할 서비스가 필요하다.

Route 53은 도메인 등록, 권한 있는 DNS, 상태 확인과 DNS 기반 라우팅을 AWS 리소스와 함께 제공한다.

이름의 `53`은 DNS가 일반적으로 사용하는 포트 번호에서 유래한다.

---

# Route 53이란?

Amazon Route 53은 가용성과 확장성을 고려해 설계된 AWS의 DNS 서비스이다.

| 기능 | 역할 |
|---|---|
| Domain Registration | 도메인 등록과 갱신 |
| DNS Service | Hosted Zone과 Record 관리 |
| Health Checking | Endpoint 상태 확인과 라우팅 연동 |

도메인을 다른 등록 기관에서 구매해도 NS 위임을 Route 53으로 설정하면 DNS만 Route 53에서 운영할 수 있다.

---

# Hosted Zone

Hosted Zone은 도메인의 DNS 레코드를 관리하는 공간이다.

예를 들어 `example.com` Hosted Zone 안에 `api.example.com` 레코드를 만든다.

Hosted Zone을 만들면 해당 Zone을 담당할 Name Server와 SOA 레코드가 생성된다.

```
Hosted Zone: example.com
├── NS    example.com
├── SOA   example.com
├── Alias api.example.com → ALB
└── TXT   example.com     → domain verification
```

도메인 등록 기관에 Route 53의 NS 값을 설정해야 인터넷의 DNS 질의가 이 Hosted Zone으로 위임된다.

## Public Hosted Zone

Public Hosted Zone은 인터넷에서 조회할 수 있는 도메인 레코드를 관리한다.

공개 웹 사이트, 모바일 API, 외부 파트너 API에 사용한다.

## Private Hosted Zone

Private Hosted Zone은 연결된 VPC 안에서만 조회할 수 있는 내부 DNS 영역이다.

`orders.internal.example.com`처럼 외부에 공개할 필요가 없는 서비스 이름에 사용한다.

| 구분 | Public Hosted Zone | Private Hosted Zone |
|---|---|---|
| 조회 위치 | 인터넷 | 연결된 VPC |
| 주요 대상 | 공개 ALB, CloudFront | Internal ALB, 내부 서비스 |
| 대표 용도 | 웹과 공개 API | MSA 내부 통신 |
| VPC 연결 | 필요 없음 | 필요 |

---

# Alias Record

Alias Record는 Route 53에서 AWS 리소스를 직접 가리키는 레코드다.

ALB, CloudFront, S3 Website Endpoint 같은 리소스에 연결할 때 사용한다.

CNAME과 달리 루트 도메인에도 사용할 수 있다.

```
api.example.com
      │ Alias
      ▼
Internet-facing ALB
      │
      ▼
Target Group
```

ALB의 IP 주소는 변경될 수 있으므로 A 레코드에 현재 IP를 직접 저장하면 안 된다.

Alias 대상에는 Route 53이 지원하는 AWS 리소스만 선택할 수 있으며 임의의 외부 도메인은 CNAME 등을 사용한다.

| 항목 | CNAME | Alias |
|---|---|---|
| 표준 DNS 타입 | 맞음 | Route 53 확장 |
| Zone Apex | 사용할 수 없음 | 사용할 수 있음 |
| AWS 리소스 선택 | 이름 직접 입력 | 지원 리소스 선택 |
| 대표 사용 | 외부 SaaS 연결 | ALB, CloudFront 연결 |

---

# 라우팅 정책

Route 53은 단순 연결 외에도 여러 라우팅 정책을 제공한다.

| 정책 | 선택 기준 | 대표 용도 |
|---|---|---|
| Simple | 별도 조건 없음 | 단일 Endpoint |
| Weighted | 설정한 가중치 | 카나리 배포 |
| Latency | 사용자와 리전 간 지연 시간 | Multi-Region |
| Failover | Primary 상태 | Active-Passive |
| Geolocation | 사용자의 지리적 위치 | 지역별 콘텐츠 |
| Geoproximity | 위치와 편향 | 지역 간 트래픽 이동 |
| Multivalue Answer | 정상인 여러 값 | 여러 Endpoint 분산 |
| IP-based | Client IP 대역 | 네트워크별 라우팅 |

라우팅 정책은 DNS 응답을 선택하며 이미 만들어진 HTTP 연결을 이동시키지 않는다.

Weighted Routing은 점진적 전환에, Latency Routing은 Multi-Region 응답 최적화에 사용한다.

Failover Routing은 Primary가 비정상일 때 Secondary를 응답하지만 TTL 때문에 모든 사용자에게 즉시 전환되지는 않는다.

---

# Health Check와 DNS 장애 조치

Route 53 Health Check는 공개 Endpoint, 계산된 상태, CloudWatch Alarm 등을 기준으로 레코드 상태를 판단할 수 있다.

```
Route 53
├── Primary Record ── Health Check: Healthy
└── Secondary Record

Primary Unhealthy
        │
        ▼
새 DNS 질의에 Secondary 응답
```

Route 53 Health Check와 [ALB Target Group Health Check](/aws-backend/part-04/06-health-check/)는 목적이 다르다.

| 구분 | Route 53 Health Check | ALB Health Check |
|---|---|---|
| 범위 | DNS Endpoint 또는 Alarm | 개별 Target |
| 결과 | DNS 응답 대상 선택 | ALB 전달 대상 선택 |
| 대표 용도 | 리전 단위 장애 조치 | 서버 단위 제외 |

ALB Alias에서 Target Health 평가 옵션을 사용하면 Alias 대상 리소스의 상태를 DNS 판단에 반영할 수 있다.

---

# AWS CLI에서는

현재 Hosted Zone 목록은 다음과 같이 확인한다.

```bash
aws route53 list-hosted-zones
```

---

# 실무에서는 어떻게 사용할까

단일 리전 서비스는 Public Hosted Zone의 Alias를 Internet-facing ALB로 연결하는 구성이 일반적이다.

내부 서비스는 Private Hosted Zone과 Internal ALB를 연결하여 공인 DNS에 내부 주소를 노출하지 않는다.

Blue/Green 환경을 서로 다른 ALB로 분리했다면 Weighted Routing으로 일부 사용자만 새 환경에 보낼 수 있다.

리전 장애 복구가 필요하면 각 리전에 독립된 데이터와 애플리케이션 계층을 준비한 뒤 Failover Routing을 적용한다.

DNS 정책만 추가한다고 데이터 동기화와 애플리케이션 복구가 자동으로 해결되는 것은 아니다.

---

# 장애 사례와 주의할 점

## Hosted Zone을 만들었지만 조회되지 않는 경우

등록 기관의 NS가 다른 Hosted Zone을 가리키거나 같은 도메인의 Hosted Zone을 중복 생성했을 가능성이 있다.

인터넷에서 실제로 위임된 NS와 작업 중인 Hosted Zone의 NS를 비교해야 한다.

## Alias 대상이 잘못된 경우

비슷한 이름의 테스트 ALB나 다른 리전의 리소스를 선택하면 잘못된 환경으로 연결된다.

레코드 변경 전 대상의 Scheme, 리전, DNS 이름을 확인한다.

## Failover가 예상보다 늦는 경우

Health Check 판정 시간과 DNS TTL, Resolver 캐시가 모두 전환 시간에 영향을 준다.

복구 목표는 DNS TTL 하나가 아니라 전체 감지와 전환 경로를 기준으로 검증해야 한다.

## Private Hosted Zone을 조회할 수 없는 경우

Hosted Zone과 VPC 연결, VPC의 DNS 지원 설정, Resolver 경로를 확인한다.

온프레미스에서 조회하려면 Route 53 Resolver Endpoint와 네트워크 연결 같은 별도 구성이 필요하다.

---

# 기억해야 할 내용

- Route 53은 AWS DNS 서비스다.
- Hosted Zone은 도메인의 DNS 설정 공간이다.
- Public Hosted Zone은 인터넷에, Private Hosted Zone은 연결된 VPC에 응답한다.
- ALB 연결에는 IP 주소 대신 Alias Record를 사용한다.
- 라우팅 정책은 새 DNS 질의에 어떤 대상을 응답할지 결정한다.
- Route 53과 ALB의 Health Check는 각각 DNS 대상과 개별 Target을 다룬다.
- Hosted Zone 생성 후 실제 NS 위임이 올바른지 확인해야 한다.

---

# 다음 Chapter

다음 Chapter에서는 Route 53이 연결하는 HTTP 진입점인 **ALB(Application Load Balancer)** 를 학습한다.

ALB의 Listener와 Rule, Reverse Proxy, Public Subnet 배치 이유를 자세히 살펴본다.


