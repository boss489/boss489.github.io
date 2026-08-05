---
title: "Chapter 01. VPC"
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 01. VPC
## AWS 안의 가상 네트워크

> **학습 목표**
>
> - VPC가 무엇인지 설명할 수 있다.
> - VPC가 왜 필요한지 이해한다.
> - VPC와 Region, Subnet의 관계를 이해한다.
> - VPC가 AWS 아키텍처에서 어떤 역할을 하는지 설명할 수 있다.

---

# 왜 VPC가 필요한가

기업의 서버는 모두 같은 네트워크에 존재하면 안 된다.

예를 들어 쇼핑몰 서비스를 생각해 보자.

- 사용자는 웹 서버에 접근해야 한다.
- 애플리케이션 서버는 데이터베이스에 접근해야 한다.
- 데이터베이스는 외부에서 직접 접근하면 안 된다.
- 운영자는 관리 서버에만 접속할 수 있어야 한다.

이처럼 서비스마다 접근 가능한 대상이 다르기 때문에 네트워크를 분리해야 한다.

온프레미스 환경에서는 스위치, 라우터, 방화벽 등을 이용해 이러한 네트워크를 구성했다.

AWS에서는 이러한 역할을 **VPC(Virtual Private Cloud)** 가 담당한다.

즉, **VPC는 AWS 안에서 나만 사용할 수 있는 독립된 네트워크 공간**이다.

---

# VPC란?

VPC(Virtual Private Cloud)는 AWS 계정 안에 생성하는 **논리적으로 격리된 가상 네트워크**이다.

EC2, ECS, RDS, ElastiCache와 같은 대부분의 AWS 리소스는 VPC 안에서 생성된다.

즉, VPC는 AWS 리소스를 실행하는 기본적인 네트워크 환경이라고 생각하면 된다.

다음 그림처럼 하나의 VPC 안에 여러 리소스가 배치된다.

![VPC overview](/assets/aws-backend/network-overview.svg)

VPC를 만들면 다음과 같은 요소들을 구성할 수 있다.

- 사용할 IP 주소 범위
- 네트워크 분리
- 인터넷 연결
- 다른 네트워크와의 연결
- 접근 제어

하지만 이러한 요소들은 모두 별도의 기능이며, 이후 Chapter에서 하나씩 자세히 학습한다.

---

# VPC의 범위

VPC는 **Region 단위의 리소스**이다.

예를 들어 서울 리전(`ap-northeast-2`)에 생성한 VPC는 서울 리전에서만 사용할 수 있다.

```
AWS
└── Seoul Region
    └── VPC
```

하나의 VPC는 여러 Availability Zone(AZ)에 걸쳐 사용할 수 있다.

즉, 하나의 VPC 안에서 AZ-a, AZ-c 등에 리소스를 배치할 수 있다.

이것이 AWS가 고가용성(Multi-AZ)을 구성하는 기본 방식이다.

---

# VPC와 Subnet의 관계

VPC는 하나의 큰 네트워크이다.

이 큰 네트워크를 실제로 사용하기 위해서는 여러 개의 작은 네트워크로 나누게 되는데, 이것이 **Subnet**이다.

```
VPC
├── Subnet
├── Subnet
├── Subnet
└── Subnet
```

즉,

- VPC는 전체 네트워크
- Subnet은 VPC를 나눈 작은 네트워크

라고 이해하면 된다.

Subnet의 종류와 구성 방법은 **다음 Chapter에서 자세히 설명한다.**

---

# VPC에서 생성되는 대표적인 리소스

실무에서는 다음과 같은 리소스들이 대부분 VPC 안에 생성된다.

| 서비스 | VPC 사용 여부 |
|---------|--------------|
| EC2 | O |
| ECS | O |
| RDS | O |
| ElastiCache | O |
| EFS | O |
| Lambda(VPC 연결 시) | O |

반면 다음과 같은 서비스는 VPC 밖에서 동작한다.

- S3
- CloudFront
- Route53
- IAM

이러한 서비스는 글로벌 서비스이거나 VPC에 종속되지 않는 서비스이다.

---

# VPC를 생성하면 무엇이 만들어질까?

새로운 VPC를 생성하면 가장 먼저 네트워크의 기본 틀이 만들어진다.

하지만 바로 서버를 실행할 수 있는 것은 아니다.

실제로 서비스를 운영하기 위해서는 추가적인 네트워크 구성이 필요하다.

대표적으로 다음과 같은 요소들을 함께 구성하게 된다.

- CIDR
- Subnet
- Route Table
- Internet Gateway
- NAT Gateway
- Security Group

이들은 모두 VPC를 중심으로 연결되는 구성 요소이며, 이후 Chapter에서 각각 자세히 살펴본다.

---

# 실무에서는 어떻게 사용할까?

Spring Boot 서비스를 AWS에 배포한다고 가정해 보자.

일반적인 구조는 다음과 같다.

```
VPC
├── ALB
├── ECS
├── RDS
└── Redis
```

모든 리소스는 동일한 VPC 안에 존재한다.

이렇게 하면 내부 리소스들은 사설 네트워크를 통해 서로 통신할 수 있으며, 외부에는 필요한 리소스만 공개할 수 있다.

실무에서는 개발, 스테이징, 운영 환경마다 VPC를 분리하거나, 하나의 계정을 여러 VPC로 나누어 운영하기도 한다.

---

# 이번 Chapter에서 기억해야 할 내용

- VPC는 AWS 안의 독립된 가상 네트워크이다.
- 대부분의 AWS 컴퓨팅 리소스는 VPC 안에서 실행된다.
- VPC는 Region 단위의 리소스이다.
- 하나의 VPC는 여러 AZ를 사용할 수 있다.
- VPC는 여러 개의 Subnet으로 나누어 사용한다.
- CIDR, Subnet, Route Table 등은 모두 VPC를 구성하는 요소이다.

---

# 다음 Chapter

다음 Chapter에서는 **CIDR(Classless Inter-Domain Routing)** 을 학습한다.

VPC를 생성할 때 가장 먼저 결정하는 **IP 주소 범위**가 무엇이며, `/16`, `/24`와 같은 표기가 어떤 의미를 가지는지 자세히 알아본다.