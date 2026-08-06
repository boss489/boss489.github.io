---
title: "Chapter 05. Internet Gateway"
permalink: /aws-backend/part-02/05-internet-gateway/
date: 2026-08-05T09:21:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. Internet Gateway
## VPC와 인터넷을 연결하는 출입구

Internet Gateway는 VPC가 인터넷과 통신할 수 있게 해주는 AWS 리소스다.

VPC에 연결하고, Route Table에서 대상으로 지정해야 실제로 동작한다.

![Route table flow](/assets/aws-backend/route-table.png)

---

# 필요한 조건

EC2가 인터넷에서 접근 가능하려면 다음 조건이 필요하다.

- VPC에 Internet Gateway가 연결되어 있어야 한다.
- Subnet Route Table에 `0.0.0.0/0 -> Internet Gateway`가 있어야 한다.
- EC2에 Public IP 또는 Elastic IP가 있어야 한다.
- Security Group에서 인바운드를 허용해야 한다.
- NACL이 트래픽을 막지 않아야 한다.

하나라도 빠지면 인터넷 접근이 되지 않는다.

---

# ALB와 Internet Gateway

실무에서는 EC2를 직접 인터넷에 노출하기보다 ALB를 Public Subnet에 둔다.

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
Spring Boot
```

이 구조에서는 애플리케이션 서버가 Public IP를 가질 필요가 없다.

---

# Internet Gateway가 하지 않는 일

Internet Gateway는 방화벽이 아니다.

다음 역할은 다른 리소스가 담당한다.

- 접근 허용/차단: Security Group, NACL
- 도메인 연결: Route53
- 로드밸런싱: ALB
- Private Subnet의 외부 요청 대행: NAT Gateway

---

# 핵심 요약

Internet Gateway는 VPC와 인터넷을 연결하는 출입구다.

하지만 실제 통신 가능 여부는 Route Table, Public IP, Security Group, NACL까지 함께 봐야 한다.
