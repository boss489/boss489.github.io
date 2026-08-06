---
title: "Chapter 11. Interview Questions"
permalink: /aws-backend/part-02/11-interview/
date: 2026-08-05T09:15:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 11. Interview Questions
## AWS Networking 면접 질문

---

# VPC

## VPC가 무엇인가요?

AWS 계정 안에 만드는 논리적으로 격리된 네트워크입니다.

EC2, RDS, ECS 같은 리소스를 배치하는 네트워크 경계 역할을 합니다.

## VPC CIDR은 왜 중요하나요?

VPC에서 사용할 수 있는 IP 범위를 결정하기 때문입니다.

너무 작게 잡으면 Subnet이나 리소스를 확장할 때 IP가 부족해질 수 있고, 다른 네트워크와 겹치면 VPC Peering이나 VPN 연결이 어려워집니다.

---

# Subnet

## Public Subnet과 Private Subnet의 차이는 무엇인가요?

Route Table의 기본 경로가 다릅니다.

Public Subnet은 `0.0.0.0/0`이 Internet Gateway를 향하고, Private Subnet은 보통 NAT Gateway를 향합니다.

## Subnet을 AZ별로 나누는 이유는 무엇인가요?

하나의 AZ 장애가 전체 서비스 장애로 이어지지 않게 하기 위해서입니다.

ALB, ECS, RDS 같은 주요 리소스를 여러 AZ에 배치하면 가용성이 높아집니다.

---

# Route Table

## Route Table은 어떤 역할을 하나요?

Subnet에서 나가는 트래픽의 다음 목적지를 결정합니다.

같은 VPC 내부 통신은 `local` Route를 사용하고, 인터넷 방향 통신은 Internet Gateway나 NAT Gateway로 보냅니다.

## Subnet 이름이 Public이면 인터넷 접근이 가능한가요?

아닙니다.

Public 여부는 이름이 아니라 Route Table, Public IP, Internet Gateway, Security Group 조건으로 결정됩니다.

---

# Internet Gateway와 NAT Gateway

## Internet Gateway와 NAT Gateway의 차이는 무엇인가요?

Internet Gateway는 VPC와 인터넷을 연결하는 출입구입니다.

NAT Gateway는 Private Subnet 리소스가 외부로 요청을 보낼 수 있게 해주는 통로입니다. 외부에서 Private Subnet으로 직접 들어오는 요청은 허용하지 않습니다.

## NAT Gateway를 사용하는 이유는 무엇인가요?

Private Subnet의 서버를 인터넷에 직접 노출하지 않으면서 외부 API 호출, 패키지 다운로드, OS 업데이트 같은 외부 요청을 처리하기 위해서입니다.

---

# Security Group과 NACL

## Security Group과 NACL의 차이는 무엇인가요?

Security Group은 리소스 단위 방화벽이고 상태 저장 방식입니다.

NACL은 Subnet 단위 접근 제어이고 상태 비저장 방식입니다.

## 백엔드 서비스에서 Security Group을 어떻게 구성하나요?

ALB는 80, 443 포트를 인터넷에 열고, Spring Boot 서버는 ALB Security Group에서 오는 8080 포트만 허용합니다.

RDS는 Spring Boot Security Group에서 오는 DB 포트만 허용합니다.

---

# Elastic IP

## Elastic IP는 언제 사용하나요?

고정 Public IP가 필요할 때 사용합니다.

백엔드에서는 NAT Gateway에 Elastic IP를 붙여 외부 API 호출 시 고정 출구 IP를 제공하는 경우가 많습니다.

---

# 장애 질문

## Private Subnet의 EC2가 외부 API를 호출하지 못하면 무엇을 확인하나요?

Route Table이 NAT Gateway를 향하는지, NAT Gateway가 Public Subnet에 있는지, NAT Gateway에 Elastic IP가 있는지, Public Subnet이 Internet Gateway로 나갈 수 있는지 확인합니다.

그 다음 Security Group과 NACL을 확인합니다.

## ALB에서 EC2로 요청이 가지 않으면 무엇을 확인하나요?

Target Group Health Check, EC2 Security Group, ALB Security Group, 애플리케이션 포트, Subnet과 Route Table을 확인합니다.

