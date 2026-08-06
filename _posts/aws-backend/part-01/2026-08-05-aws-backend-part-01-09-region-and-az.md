---
title: "Chapter 09. Region\uACFC Availability Zone"
permalink: /aws-backend/part-01/09-region-and-az/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 09. Region과 Availability Zone
## 가용성을 만드는 기본 단위

> **학습 목표**
>
> - Region과 AZ의 차이를 설명할 수 있다.
> - Multi-AZ 구성이 필요한 이유를 이해한다.
> - 지연 시간과 장애 격리 관점에서 Region 선택을 설명할 수 있다.

---

# Region과 AZ의 차이

Region은 AWS 리소스를 배치하는 지리적 단위다.

AZ는 Region 안에서 독립적으로 장애가 격리되는 데이터센터 그룹이다.

```
Region
├── AZ 1
├── AZ 2
└── AZ 3
```

---

# 왜 여러 AZ를 사용하는가

하나의 AZ에만 서버와 데이터베이스를 두면 AZ 장애 시 전체 서비스가 멈출 수 있다.

Multi-AZ 구성은 리소스를 여러 AZ에 나누어 배치해 이 위험을 줄인다.

대표 예시는 다음과 같다.

- ALB를 2개 이상의 AZ에 배치
- ECS 또는 EC2를 여러 AZ에 배치
- RDS Multi-AZ 사용
- Subnet을 AZ별로 분리

---

# Region 선택 기준

Region은 아무 곳이나 선택하지 않는다.

다음 기준을 고려한다.

- 주요 사용자와 가까운가?
- 필요한 AWS 서비스가 지원되는가?
- 데이터 보관 규정에 맞는가?
- 비용이 적절한가?
- 다른 시스템과 네트워크 지연 시간이 낮은가?

---

# 기억해야 할 내용

- Region은 지리적 위치이고 AZ는 장애 격리 단위다.
- 고가용성은 여러 AZ를 사용하는 것에서 시작한다.
- Region 선택은 지연 시간, 규정, 비용, 지원 서비스를 함께 고려한다.


