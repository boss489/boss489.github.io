---
title: "Chapter 14. Interview Questions"
permalink: /aws-backend/part-01/14-interview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 14. Interview Questions
## AWS Foundation 면접 질문

---

# Cloud

## Cloud Computing이 왜 등장했나요?

물리 서버를 직접 구매하고 운영하는 방식은 비용이 크고 증설이 느렸습니다.

Cloud는 필요한 리소스를 필요한 시점에 만들고 사용한 만큼 비용을 내는 방식으로 이 문제를 줄였습니다.

## On-Premise와 Cloud의 가장 큰 차이는 무엇인가요?

On-Premise는 인프라를 직접 소유하고 운영합니다.

Cloud는 인프라를 직접 소유하지 않고 API로 생성해 사용합니다.

---

# Virtualization

## Virtual Machine은 무엇인가요?

물리 서버 위에서 소프트웨어로 만든 가상 서버입니다.

각 VM은 독립된 OS와 자원을 가진 것처럼 동작합니다.

## Hypervisor는 어떤 역할을 하나요?

물리 서버의 CPU, 메모리, 디스크, 네트워크 자원을 여러 VM에 나누어 주고 VM 실행을 관리합니다.

---

# AWS Infrastructure

## Region과 AZ의 차이는 무엇인가요?

Region은 AWS 리소스를 배치하는 지리적 영역입니다.

AZ는 Region 안에서 장애가 격리되는 데이터센터 그룹입니다.

## Multi-AZ를 사용하는 이유는 무엇인가요?

하나의 AZ에 장애가 발생해도 다른 AZ에서 서비스를 계속하기 위해서입니다.

ALB, EC2/ECS, RDS를 여러 AZ에 배치하면 가용성이 높아집니다.

## Edge Location은 무엇인가요?

사용자 가까이에서 CloudFront, Route 53 같은 네트워크 기능을 제공하는 AWS 거점입니다.

정적 콘텐츠 캐싱이나 DNS 응답에 사용됩니다.

---

# Responsibility

## Shared Responsibility Model이란 무엇인가요?

AWS와 사용자가 보안 책임을 나누어 가지는 모델입니다.

AWS는 Cloud 자체의 보안을 책임지고, 사용자는 Cloud 안에서 만든 리소스, 권한, 데이터, 애플리케이션 보안을 책임집니다.

## EC2를 사용할 때 사용자의 보안 책임은 무엇인가요?

Security Group, IAM, OS 패치, 애플리케이션 보안, 데이터 암호화, 로그 설정 등이 사용자의 책임입니다.

---

# 심화 질문

## Managed Service를 사용하면 운영 책임이 모두 사라지나요?

아닙니다.

AWS가 일부 운영을 대신하지만 권한, 데이터 모델, 네트워크 접근, 비용, 장애 대응 설계는 여전히 사용자의 책임입니다.


