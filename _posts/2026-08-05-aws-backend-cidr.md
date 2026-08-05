---
title: "Chapter 02. CIDR"
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. CIDR
## IP 범위를 표현하는 방법

CIDR(Classless Inter-Domain Routing)은 IP 주소 범위를 표현하는 방식이다.

AWS에서는 VPC와 Subnet의 IP 대역을 CIDR로 지정한다.

![CIDR and subnet split](/assets/aws-backend/cidr-subnet.svg)

---

# 기본 예시

```
10.0.0.0/16
```

`/16`은 앞의 16비트가 네트워크 주소라는 뜻이다.

범위는 다음과 같다.

```
10.0.0.0 ~ 10.0.255.255
```

즉 약 65,536개의 IP 주소를 표현한다.

---

# 자주 쓰는 CIDR

| CIDR | IP 개수 | 예시 |
|---|---:|---|
| `/16` | 65,536 | VPC |
| `/20` | 4,096 | 큰 Subnet |
| `/24` | 256 | 일반 Subnet |
| `/28` | 16 | 작은 Subnet |

AWS는 Subnet마다 IP 5개를 예약한다.

예를 들어 `/24` Subnet은 256개처럼 보이지만 실제 사용 가능한 IP는 251개다.

---

# 사설 IP 대역

VPC에서는 보통 사설 IP 대역을 사용한다.

| 범위 | 설명 |
|---|---|
| `10.0.0.0/8` | 가장 넓은 사설 대역 |
| `172.16.0.0/12` | 중간 크기 사설 대역 |
| `192.168.0.0/16` | 작은 네트워크에서 자주 사용 |

실무에서는 `10.0.0.0/16` 같은 대역을 많이 사용한다.

---

# 설계 예시

```
VPC: 10.0.0.0/16

Public Subnet A:  10.0.1.0/24
Public Subnet C:  10.0.2.0/24
Private Subnet A: 10.0.11.0/24
Private Subnet C: 10.0.12.0/24
DB Subnet A:      10.0.21.0/24
DB Subnet C:      10.0.22.0/24
```

숫자에 의미를 주면 운영 중에 알아보기 쉽다.

---

# 주의할 점

- 다른 VPC, 사내망, VPN 대역과 겹치면 안 된다.
- 너무 작은 CIDR은 나중에 Subnet을 추가하기 어렵다.
- IP가 부족하면 ECS Task, EC2, RDS 확장에 문제가 생긴다.

---

# 핵심 요약

CIDR은 네트워크의 IP 범위를 정하는 표기법이다.

VPC는 크게 잡고, Subnet은 역할과 AZ 단위로 나누는 것이 기본이다.
