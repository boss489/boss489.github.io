---
title: "Chapter 05. ENI"
permalink: /aws-backend/part-03/05-eni/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 05. ENI
## EC2의 네트워크 인터페이스

> **학습 목표**
>
> - ENI의 역할을 설명할 수 있다.
> - Private IP, Public IP, Security Group과의 관계를 이해한다.
> - EC2 네트워크 문제를 어떤 관점으로 봐야 하는지 이해한다.

---

# ENI란?

ENI(Elastic Network Interface)는 EC2에 연결되는 가상 네트워크 카드다.

EC2가 VPC 안에서 통신하려면 ENI가 필요하다.

ENI에는 다음이 연결된다.

- Private IP
- Public IP 또는 Elastic IP
- Security Group
- Subnet
- MAC Address

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# Private IP와 Public IP

Private IP는 VPC 내부 통신에 사용한다.

Public IP는 인터넷에서 접근할 때 사용한다.

EC2가 Public Subnet에 있어도 Public IP가 없으면 인터넷에서 직접 접근할 수 없다.

---

# Security Group

Security Group은 ENI에 적용된다.

즉 EC2에 방화벽이 붙는다고 표현하지만, 실제로는 EC2의 네트워크 인터페이스에 규칙이 적용된다고 이해하면 좋다.

---

# 기억해야 할 내용

- ENI는 EC2의 네트워크 카드다.
- EC2 통신 문제는 ENI, Subnet, Route Table, Security Group을 함께 봐야 한다.
- Public IP가 없으면 인터넷에서 직접 접근할 수 없다.
- Security Group은 리소스의 네트워크 인터페이스에 적용된다.


