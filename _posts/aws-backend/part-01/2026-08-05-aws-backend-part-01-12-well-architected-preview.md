---
title: "Chapter 12. Well-Architected Framework Preview"
permalink: /aws-backend/part-01/12-well-architected-preview/
date: 2026-08-05T09:25:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Chapter 12. Well-Architected Framework Preview
## AWS가 권장하는 아키텍처 관점

> **학습 목표**
>
> - Well-Architected Framework의 목적을 이해한다.
> - 6가지 Pillar를 설명할 수 있다.
> - 백엔드 설계에서 어떤 질문을 던져야 하는지 이해한다.

---

# Well-Architected Framework란?

Well-Architected Framework는 AWS에서 안정적이고 효율적인 시스템을 설계하기 위한 기준이다.

정답 아키텍처 하나를 제시하는 것이 아니라, 좋은 설계를 검토하기 위한 질문 모음에 가깝다.

---

# 6가지 Pillar

| Pillar | 질문 |
|---|---|
| Operational Excellence | 운영과 배포를 어떻게 안정화할 것인가 |
| Security | 권한과 데이터를 어떻게 보호할 것인가 |
| Reliability | 장애가 나도 어떻게 복구할 것인가 |
| Performance Efficiency | 필요한 성능을 어떻게 낼 것인가 |
| Cost Optimization | 낭비되는 비용을 어떻게 줄일 것인가 |
| Sustainability | 자원을 얼마나 효율적으로 사용할 것인가 |

---

# 백엔드 관점의 예시

Spring Boot 서비스를 운영한다면 다음 질문을 해야 한다.

- 배포 실패 시 롤백할 수 있는가?
- ALB에서 애플리케이션 Health Check를 확인하는가?
- RDS는 Multi-AZ인가?
- Secrets를 코드에 넣지 않았는가?
- CloudWatch 로그로 장애를 추적할 수 있는가?
- 트래픽이 적은 시간에도 과도한 리소스를 켜두고 있지 않은가?

---

# 기억해야 할 내용

- Well-Architected는 설계 검토 기준이다.
- 6가지 Pillar는 운영, 보안, 신뢰성, 성능, 비용, 지속 가능성이다.
- 이후 Part에서 배우는 서비스들은 이 기준으로 다시 평가할 수 있다.


