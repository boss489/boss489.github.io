---
title: "Chapter 10. Part 2 Summary"
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 10. Part 2 Summary
## AWS Networking 핵심 정리

Part 2의 핵심은 VPC 안에서 트래픽 흐름을 설명할 수 있는 것이다.

![VPC overview](/assets/aws-backend/network-overview.svg)

---

# 전체 구조

```
Internet
  │
Internet Gateway
  │
Public Subnet
  │
ALB
  │
Private Subnet
  │
ECS / EC2
  │
RDS / Redis
```

---

# 핵심 개념

| 개념 | 역할 |
|---|---|
| VPC | AWS 리소스를 담는 네트워크 경계 |
| CIDR | IP 범위를 표현하는 방식 |
| Subnet | VPC를 역할과 AZ 기준으로 나누는 단위 |
| Route Table | 트래픽의 다음 목적지를 결정 |
| Internet Gateway | VPC와 인터넷 연결 |
| NAT Gateway | Private Subnet의 외부 요청 통로 |
| Security Group | 리소스 단위 방화벽 |
| NACL | Subnet 단위 접근 제어 |
| Elastic IP | 고정 Public IP |

---

# Public과 Private의 기준

Public Subnet:

```
0.0.0.0/0 -> Internet Gateway
```

Private Subnet:

```
0.0.0.0/0 -> NAT Gateway
```

Subnet의 이름이 아니라 Route Table이 기준이다.

---

# 실무 기본 설계

- ALB는 Public Subnet에 둔다.
- Spring Boot 서버는 Private Subnet에 둔다.
- RDS와 Redis는 Private Subnet에 둔다.
- 외부 API 호출이 필요하면 NAT Gateway를 둔다.
- 접근 제어는 Security Group 중심으로 한다.

---

# 장애 확인 순서

네트워크가 안 될 때는 아래 순서로 본다.

1. 대상 리소스가 올바른 Subnet에 있는가?
2. Route Table이 올바른 대상으로 향하는가?
3. Internet Gateway 또는 NAT Gateway가 연결되어 있는가?
4. Security Group이 허용하는가?
5. NACL이 막고 있지 않은가?
6. Public IP 또는 Elastic IP가 필요한 구조인가?

---

# 핵심 요약

AWS 네트워크 문제는 대부분 Route Table, Security Group, Subnet 배치에서 발생한다.

구조를 그림으로 그릴 수 있으면 장애 분석도 빨라진다.
