---
title: "Chapter 02. EC2"
permalink: /aws-backend/part-03/02-ec2/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 02. EC2
## AWS에서 사용하는 가상 서버

> **학습 목표**
>
> - EC2가 무엇인지 설명할 수 있다.
> - 인스턴스 타입과 상태 변화를 이해한다.
> - EC2가 VPC, AMI, EBS, ENI와 연결되는 방식을 설명할 수 있다.

---

# EC2란?

EC2(Elastic Compute Cloud)는 AWS에서 제공하는 가상 서버 서비스다.

사용자는 CPU, 메모리, 네트워크 성능에 맞는 인스턴스 타입을 선택해 서버를 생성한다.

EC2는 다음 요소와 함께 만들어진다.

- AMI
- Instance Type
- EBS Volume
- ENI
- Security Group
- Key Pair

![EC2 anatomy](/assets/aws-backend/ec2-anatomy.png)

---

# 인스턴스 타입

인스턴스 타입은 EC2의 CPU, 메모리, 네트워크 성능을 결정한다.

예시는 다음과 같다.

| 계열 | 용도 |
|---|---|
| t, m | 일반 목적 |
| c | CPU 중심 |
| r | 메모리 중심 |
| i | 스토리지 성능 중심 |
| g | GPU 중심 |

Spring Boot 일반 API 서버는 처음에는 `t` 또는 `m` 계열로 시작하는 경우가 많다.

---

# 상태 변화

EC2는 상태를 가진다.

```
pending -> running -> stopping -> stopped -> terminated
```

`stopped` 상태에서는 EBS 루트 볼륨이 남아 있을 수 있다.

`terminated` 상태는 인스턴스를 삭제한 상태다.

---

# 기억해야 할 내용

- EC2는 가상 서버다.
- 인스턴스 타입은 서버 성능을 결정한다.
- EC2는 AMI로 시작하고 EBS와 ENI가 붙는다.
- EC2를 인터넷에 노출할지는 Subnet, Route Table, Public IP, Security Group이 결정한다.


