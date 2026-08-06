---
title: "Chapter 07. AWS \uC18C\uAC1C"
permalink: /aws-backend/part-01/07-introduction-to-aws/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 07. AWS 소개
## 인프라를 API로 제공하는 플랫폼

> **학습 목표**
>
> - AWS가 어떤 플랫폼인지 설명할 수 있다.
> - AWS Console과 AWS CLI의 차이를 이해한다.
> - AWS 서비스를 큰 분류로 나눌 수 있다.

---

# AWS란?

AWS(Amazon Web Services)는 컴퓨팅, 스토리지, 네트워크, 데이터베이스 같은 인프라 기능을 서비스로 제공하는 Cloud 플랫폼이다.

사용자는 서버실을 직접 운영하지 않고 AWS 리소스를 생성해 서비스를 구성한다.

---

# AWS 서비스 분류

대표적인 서비스 분류는 다음과 같다.

| 분류 | 대표 서비스 |
|---|---|
| Compute | EC2, ECS, Lambda |
| Network | VPC, Route 53, CloudFront |
| Storage | S3, EBS, EFS |
| Database | RDS, DynamoDB, ElastiCache |
| Security | IAM, KMS, WAF |
| Observability | CloudWatch, CloudTrail |

---

# AWS Console

AWS Console은 브라우저에서 리소스를 생성하고 관리하는 화면이다.

처음 학습할 때는 Console이 구조를 이해하기 좋다.

하지만 실무에서는 수동 클릭만으로 운영하면 변경 이력을 관리하기 어렵다.

---

# AWS CLI

AWS CLI는 터미널에서 AWS API를 호출하는 도구다.

예를 들어 S3 버킷 목록은 다음처럼 확인한다.

```bash
aws s3 ls
```

실무에서는 CLI, SDK, Terraform, CloudFormation 같은 자동화 도구를 함께 사용한다.

---

# 기억해야 할 내용

- AWS는 인프라 기능을 API로 제공하는 플랫폼이다.
- Console은 학습과 확인에 좋고, CLI는 자동화에 좋다.
- AWS 서비스는 Compute, Network, Storage, Database, Security로 나누어 이해하면 쉽다.


