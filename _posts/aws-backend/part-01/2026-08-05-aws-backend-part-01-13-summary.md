---
title: "Chapter 13. Part 1 Summary"
permalink: /aws-backend/part-01/13-summary/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 13. Part 1 Summary
## AWS Foundation 핵심 정리

Part 1의 핵심은 Cloud가 왜 등장했고, AWS 인프라가 어떤 기본 단위로 구성되는지 설명하는 것이다.

![Cloud evolution](/assets/aws-backend/cloud-evolution.png)

---

# 핵심 개념

| 개념 | 설명 |
|---|---|
| On-Premise | 직접 서버와 네트워크를 소유하고 운영 |
| Virtualization | 물리 서버를 여러 VM으로 나누는 기술 |
| Hypervisor | VM을 만들고 자원을 배분하는 계층 |
| Cloud Computing | 필요한 리소스를 네트워크로 사용하는 방식 |
| Region | AWS 리소스를 배치하는 지리적 영역 |
| AZ | Region 안의 장애 격리 단위 |
| Edge Location | 사용자 가까이에서 네트워크 기능 제공 |
| Shared Responsibility | AWS와 사용자가 보안 책임을 나누는 모델 |

---

# 전체 흐름

```
On-Premise
  -> Virtualization
  -> Cloud Computing
  -> AWS
  -> Region
  -> AZ
  -> Edge Network
  -> Shared Responsibility
```

---

# 실무 포인트

- Cloud는 서버를 없앤 것이 아니라 운영 모델을 바꾼 것이다.
- EC2는 가상화된 컴퓨팅 리소스이다.
- 고가용성 설계는 Multi-AZ에서 시작한다.
- 보안 설정은 사용자의 책임으로 남는다.
- AWS 서비스는 책임 범위를 줄여주지만 설계 책임을 없애지는 않는다.

---

# 다음 Part 예고

Part 2에서는 AWS 리소스가 서로 통신하는 네트워크 구조를 학습한다.

VPC, Subnet, Route Table, Internet Gateway, NAT Gateway, Security Group을 이해해야 실제 백엔드 아키텍처를 설명할 수 있다.


