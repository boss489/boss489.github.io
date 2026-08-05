---
title: "Chapter 04. Route Table"
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 04. Route Table
## 트래픽의 다음 목적지를 정하는 규칙

Route Table은 Subnet에서 나가는 트래픽이 어디로 가야 하는지 결정한다.

Subnet이 Public인지 Private인지는 이름이 아니라 Route Table로 결정된다.

![Route table flow](/assets/aws-backend/route-table.svg)

---

# 기본 구조

Route Table은 목적지와 대상의 목록이다.

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | Internet Gateway |

`local`은 같은 VPC 내부 통신을 뜻한다.

`0.0.0.0/0`은 나머지 모든 IPv4 트래픽을 뜻한다.

---

# Public Subnet Route

Public Subnet은 인터넷 방향 Route가 Internet Gateway로 향한다.

```
10.0.0.0/16  -> local
0.0.0.0/0    -> igw-xxxx
```

이 Route가 있어야 외부 인터넷과 통신할 수 있다.

---

# Private Subnet Route

Private Subnet은 인터넷 방향 Route가 NAT Gateway로 향한다.

```
10.0.0.0/16  -> local
0.0.0.0/0    -> nat-xxxx
```

이 구조에서는 외부에서 Private Subnet으로 직접 들어올 수 없다.

Private Subnet의 서버가 외부로 요청을 보낼 수만 있다.

---

# 트래픽 흐름 예시

Spring Boot 서버가 외부 결제 API를 호출하는 경우:

```
ECS Task
  │
Private Subnet Route Table
  │
NAT Gateway
  │
Internet Gateway
  │
External API
```

사용자가 API를 호출하는 경우:

```
User
  │
Internet Gateway
  │
ALB
  │
ECS Task
```

---

# 핵심 요약

Route Table은 Subnet의 트래픽 방향을 결정한다.

Public과 Private의 차이는 Subnet 이름이 아니라 `0.0.0.0/0` Route가 어디를 향하는지다.
