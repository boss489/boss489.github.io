---
title: "Chapter 03. Subnet"
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 03. Subnet
## VPC를 나누는 단위

Subnet은 VPC 안의 IP 범위를 더 작게 나눈 네트워크다.

AWS 리소스는 VPC에 직접 배치되는 것이 아니라 보통 특정 Subnet에 배치된다.

![CIDR and subnet split](/assets/aws-backend/cidr-subnet.svg)

---

# 왜 나누는가

Subnet을 나누는 이유는 역할과 장애 범위를 분리하기 위해서다.

- Public Subnet: 인터넷과 직접 연결되는 리소스
- Private Subnet: 애플리케이션 서버
- DB Subnet: 데이터베이스

또한 Subnet은 하나의 Availability Zone에 속한다.

그래서 고가용성을 위해 최소 2개 AZ에 Subnet을 만든다.

---

# Public Subnet

Public Subnet은 인터넷으로 나가는 Route가 Internet Gateway를 향하는 Subnet이다.

보통 다음 리소스를 둔다.

- ALB
- NAT Gateway
- Bastion Host

Public Subnet이라고 해서 모든 리소스가 자동으로 인터넷에 공개되는 것은 아니다.

Public IP, Route Table, Security Group이 함께 맞아야 외부 접근이 가능하다.

---

# Private Subnet

Private Subnet은 인터넷에서 직접 접근할 수 없는 Subnet이다.

보통 다음 리소스를 둔다.

- EC2
- ECS Task
- RDS
- ElastiCache

Private Subnet의 서버가 외부 API나 패키지 저장소에 접근해야 하면 NAT Gateway를 사용한다.

---

# 설계 예시

```
VPC 10.0.0.0/16

ap-northeast-2a
├── Public Subnet  10.0.1.0/24
├── Private Subnet 10.0.11.0/24
└── DB Subnet      10.0.21.0/24

ap-northeast-2c
├── Public Subnet  10.0.2.0/24
├── Private Subnet 10.0.12.0/24
└── DB Subnet      10.0.22.0/24
```

---

# 핵심 요약

Subnet은 VPC를 역할과 AZ 기준으로 나누는 단위다.

외부 진입점은 Public Subnet에, 애플리케이션과 데이터는 Private Subnet에 두는 것이 기본이다.
