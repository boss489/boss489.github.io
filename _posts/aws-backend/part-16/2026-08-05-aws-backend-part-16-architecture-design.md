---
title: "Part 16. Architecture Design"
permalink: /aws-backend/part-16/
date: 2026-08-05T09:01:00+09:00
categories:
  - aws
tags:
  - aws
  - backend
  - study
last_modified_at: 2026-08-05T09:00:00+09:00
---

# Part 16. Architecture Design
## 실제 서비스 아키텍처 설계

> 목표
>
> 요구사항과 제약에서 출발해 쇼핑몰 서비스를 AWS에서 운영할 수 있는 구조로 설계한다.

---

# 학습 내용

| Chapter | 핵심 질문 |
|---|---|
| [01. Design Requirements](/aws-backend/part-16/01-design-requirements/) | 무엇을 어느 수준까지 보장해야 하는가 |
| [02. Shopping Mall Architecture](/aws-backend/part-16/02-shopping-mall-architecture/) | 실제 요청과 네트워크 경로를 어떻게 배치하는가 |
| [03. REST API Design](/aws-backend/part-16/03-rest-api-design/) | 재시도에도 안전한 HTTP 계약을 어떻게 만드는가 |
| [04. Data Architecture](/aws-backend/part-16/04-data-architecture/) | 주문과 결제의 일관성 경계를 어디에 두는가 |
| [05. Cache Consistency](/aws-backend/part-16/05-cache-consistency/) | Redis 성능과 정합성을 어떻게 조정하는가 |
| [06. Event-Driven Architecture](/aws-backend/part-16/06-event-driven-architecture/) | 중복과 순서와 재처리를 어떻게 다루는가 |
| [07. Batch Architecture](/aws-backend/part-16/07-batch-architecture/) | 배치를 어떻게 재시작 가능하고 멱등하게 만드는가 |
| [08. File Delivery Architecture](/aws-backend/part-16/08-file-delivery-architecture/) | 파일 업로드와 검증과 전송을 어떻게 분리하는가 |
| [09. Scalability and High Availability](/aws-backend/part-16/09-scalability-high-availability/) | 병목을 보호하면서 어떻게 수평 확장하는가 |
| [10. Disaster Recovery](/aws-backend/part-16/10-disaster-recovery/) | Region 장애에서 목표 시간과 데이터로 어떻게 복구하는가 |
| [11. Architecture Review](/aws-backend/part-16/11-architecture-review/) | 출시 전에 어떤 증거와 운영 준비를 확인하는가 |
| [12. Part 16 Summary](/aws-backend/part-16/12-summary/) | 전체 설계 결정을 어떻게 연결하는가 |
| [13. Interview Questions](/aws-backend/part-16/13-interview/) | 설계 근거를 어떻게 짧고 정확하게 설명하는가 |

---

# 완료 후 설명할 수 있어야 하는 것

- 요구사항, 제약, 대안, 결정과 검증으로 이어지는 설계 과정
- 사용자 요청, API, 이벤트, 배치와 파일 서비스의 서로 다른 실제 흐름
- Redis, Aurora, 메시징 서비스와 S3를 선택하고 배치하는 기준
- Auto Scaling과 Multi-AZ를 적용할 위치와 데이터베이스 병목
- Multi-AZ와 DR의 차이 및 RTO와 RPO에 따른 복구 전략

---

다음 Part

→ [Part 17. Final Project](/aws-backend/part-17/)

