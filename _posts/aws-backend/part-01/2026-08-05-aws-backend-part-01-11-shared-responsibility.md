---
title: "Chapter 11. Shared Responsibility Model"
permalink: /aws-backend/part-01/11-shared-responsibility/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 11. Shared Responsibility Model
## AWS와 사용자가 나누어 가지는 보안 책임

> **학습 목표**
>
> - Security of the Cloud와 Security in the Cloud의 차이를 설명할 수 있다.
> - AWS가 책임지는 영역과 사용자가 책임지는 영역을 구분할 수 있다.
> - IAM, Patch, Encryption이 왜 사용자 책임에 포함되는지 이해한다.

---

# Shared Responsibility Model

Cloud를 사용해도 보안 책임이 모두 AWS로 넘어가지 않는다.

AWS는 Cloud 자체의 보안을 책임지고, 사용자는 Cloud 안에서 만든 리소스와 데이터의 보안을 책임진다.

![Shared responsibility](/assets/aws-backend/shared-responsibility.png)

---

# AWS의 책임

AWS는 Security of the Cloud를 책임진다.

예시는 다음과 같다.

- 데이터센터 물리 보안
- 서버 하드웨어
- 글로벌 네트워크
- 관리형 서비스의 기반 인프라
- 리전과 AZ 시설 운영

---

# 사용자의 책임

사용자는 Security in the Cloud를 책임진다.

예시는 다음과 같다.

- IAM 권한
- Security Group 설정
- 애플리케이션 취약점
- 데이터 암호화 설정
- OS 패치
- 로그와 감사 설정

---

# 서비스에 따라 달라지는 책임

EC2를 사용하면 OS 패치와 런타임 설치는 사용자의 책임이다.

RDS를 사용하면 데이터베이스 엔진 운영 일부는 AWS가 관리하지만, 계정 권한과 데이터 접근 제어는 사용자가 관리한다.

S3를 사용해도 버킷 정책을 잘못 열면 데이터가 노출될 수 있다.

---

# 기억해야 할 내용

- AWS를 쓴다고 보안 책임이 사라지지 않는다.
- AWS는 Cloud 자체를 보호한다.
- 사용자는 Cloud 안의 설정, 권한, 데이터, 애플리케이션을 보호한다.
- 서비스 모델에 따라 책임 범위가 달라진다.


