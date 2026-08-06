---
title: "Chapter 08. AWS Global Infrastructure"
permalink: /aws-backend/part-01/08-global-infrastructure/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 08. AWS Global Infrastructure
## 전 세계에 분산된 AWS 인프라

> **학습 목표**
>
> - Region, Availability Zone, Edge Location의 관계를 설명할 수 있다.
> - AWS가 인프라를 전 세계에 나누어 배치하는 이유를 이해한다.
> - 글로벌 인프라가 가용성과 지연 시간에 주는 영향을 설명할 수 있다.

---

# 글로벌 인프라가 필요한 이유

사용자는 전 세계에 있다.

한국 사용자가 미국 서버에 접속하면 지연 시간이 커질 수 있다.

또한 하나의 데이터센터 장애가 전체 서비스 장애로 이어지면 안 된다.

AWS는 이를 해결하기 위해 Region, AZ, Edge Location 구조를 사용한다.

![AWS global infrastructure](/assets/aws-backend/aws-global-infra.png)

---

# Region

Region은 AWS 인프라가 위치한 지리적 영역이다.

예를 들어 서울 리전은 `ap-northeast-2`이다.

서비스를 만들 때는 사용자 위치, 법적 요구사항, 비용, 지원 서비스 범위를 고려해 Region을 선택한다.

---

# Availability Zone

AZ는 Region 안에 있는 독립된 데이터센터 그룹이다.

하나의 Region은 여러 AZ로 구성된다.

Multi-AZ 구성은 특정 AZ 장애가 발생해도 다른 AZ에서 서비스를 계속하기 위한 기본 전략이다.

---

# Local Zone과 Wavelength

Local Zone은 특정 도시 근처에 컴퓨팅 자원을 배치해 지연 시간을 줄이는 구조다.

Wavelength는 통신사 5G 네트워크에 AWS 컴퓨팅 자원을 배치하는 구조다.

일반적인 백엔드 서비스에서는 먼저 Region과 AZ를 이해하면 충분하다.

---

# 기억해야 할 내용

- Region은 지리적 영역이다.
- AZ는 Region 안의 장애 격리 단위다.
- Edge Location은 사용자 가까이에서 콘텐츠나 네트워크 기능을 제공한다.
- 기본 백엔드 설계는 Multi-AZ 배치를 전제로 한다.


